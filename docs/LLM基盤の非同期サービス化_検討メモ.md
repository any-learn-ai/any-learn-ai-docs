# LLM 基盤の非同期マイクロサービス化 — 検討メモ

**作成: 2026-08-07 / 状態: 設計待ち（実装は未着手）**

このメモは、次に着手する人が **設計から始められる**ように、
2026-08-07 の検討で出た事実・実測値・未解決の論点をまとめたもの。

⚠️ ここに書いてある数値は推測ではなく、その日に dev で全経路を流して
測ったもの。設計の根拠にそのまま使ってよい。

---

## 1. なぜやるのか

### きっかけ

assessments が Luna（OpenAI 互換）を呼べない。`src/shared/claude.ts` が
Anthropic 専用（`new Anthropic({apiKey})`）で、OpenAI 互換プロバイダを
持っていないため。core 側でやったモデル差し替え（プログラム生成を
Sonnet → Luna で 54.71円/209.6秒 → 2.87円/77.7秒）が、assessments には
一切及んでいない。

### 見つかった構造的な問題

| 問題 | 実害 |
|---|---|
| LLM 層が core と assessments に重複 | assessments は Anthropic 専用のまま取り残された |
| 登録簿（使えるモデル）と単価表が別ファイル | **2回**ずれた。qwen3.7-plus で原価18倍表示、gpt-5.6-luna で全処理が Opus 単価表示 |
| API キーの読み取り権が **23 個の Lambda** に | `get-session` `list-goals` `delete-program` など LLM を呼ばないものまで持つ |
| 大きな Lambda（1〜3GB）がネットワーク待ちにフル課金 | 費用の6割が「待つだけ」の時間 |

---

## 2. 実測値（2026-08-07 / dev / 全用途 gpt-5.6-luna）

### 所要時間

| 用途 | モデル | 実測 | 1プログラムでの回数 | Lambda メモリ |
|---|---|---|---|---|
| 目標分析 | Luna | 5.6 秒 | 1 | 1769 MB |
| 目標チャット | Luna | 3.2 秒 | 数回 | 1024 MB |
| 計画の見積もり | Luna | 10.9 秒 | 1 | 1769 MB |
| 学習計画の算出 | — | 5 秒（AI 呼ばず） | 1 | 1769 MB |
| プログラム生成 | Luna | **59.4 秒** | 1 | **3008 MB** |
| 設問生成 | Sonnet | **73 秒** | 1 | 1769 MB |
| 診断レポート | Sonnet | **123 秒** | 1 | 1769 MB |
| 解説 | Luna | 9.8 秒 | **75** | 1536 MB |
| 確認問題 | Luna | 6.8 秒 | 75 | 1024 MB |
| 採点 | Luna | 3.6 秒 | 75 | 1024 MB |
| Q&A | Luna | 3.3 秒 | 20 | 1024 MB |

LLM 呼び出しは **1人あたり約 262 回**。

### 費用（100人が1プログラム＝25セッションを完走）

| | 現在 | 全非同期サービス化 |
|---|---|---|
| Lambda | 655 円 | **273 円** |
| ポーリング（API GW + Lambda） | 50 円 | 0 円（廃止） |
| WebSocket | — | +8 円 |
| **合計** | **705 円** | **281 円（−60%）** |
| （参考）AI 呼び出しの原価 | 約 3,000〜4,000 円 | 同じ |

削減の正体は「1〜3GB の Lambda がネットワークを待つ時間」。
待つ役を 512MB の LLM サービスへ移すだけで消える。

### 単価（ap-northeast-1 / AWS Price List API で取得）

- API Gateway REST: **$4.25** / 100万リクエスト
- WebSocket メッセージ: **$1.26** / 100万通
- WebSocket 接続時間: **$0.315** / 100万接続分
- Lambda: **$0.0000166667** / GB秒
- SQS: $0.40 / 100万件、EventBridge: $1.00 / 100万件

---

## 3. 影響範囲

### LLM を呼んでいる 12 ファイル

```
core/src/handlers/evaluate-answer.ts      core/src/workers/analyze-goal.ts
core/src/handlers/goal-chat.ts            core/src/workers/explanation-worker.ts
core/src/handlers/micro-step-qa.ts        core/src/workers/generate-program.ts
core/src/handlers/session-question.ts     assessments/src/workers/generate-assessment.ts
core/src/domain/plan-estimate.ts          assessments/src/workers/generate-report.ts
core/src/domain/quiz-generation.ts        assessments/src/handlers/submit-answer.ts
```

各々が「振り分け」と「後処理」に割れる。Lambda は **12 → 約30** に増える。

### 後処理へ移る処理

LLM 呼び出しの**直後**にある以下が、別 Lambda に移る。

- 設問数の検証と `trimToRange`（assessments）
- プログラム構造の検証と作り直しの判定（core）
- `saveGeneratedQuestions` / `completeProgramGeneration` / `saveExplanation`
- 確認問題の先読み（`prefetchQuiz`）
- **品質検査（`inspect`）と再生成の判断** ← 下記の論点を参照

### フロントで同期応答を前提にしている 5 か所

```
app/goals/new/page.tsx:150          sendGoalChat
app/learn/[sessionId]/page.tsx:578  askMicroStepQuestion
app/learn/[sessionId]/page.tsx:647  generateQuiz
app/learn/[sessionId]/page.tsx:669  evaluateAnswer
app/learn/[sessionId]/page.tsx:787  askSessionQuestion
```

これらを非同期にするには WebSocket（または継続ポーリング）が要る。
`lib-web/poll.ts` と `lib-web/use-wait-progress.ts` も作り直しになる。

---

## 4. 未解決の論点（設計で決めること）

### ⚠️ 論点1: 品質検査の再生成ループをどこに置くか — **最大の争点**

現在 `generateStructured` はこうなっている。

```
生成 → inspect（必須項目の欠落・簡体字の混入・締め文句の誤り）
     → 不備があれば もう一度生成 → 再検査
     → それでも critical なら失敗
```

この「もう一度生成」が非同期境界をまたぐ。しかも検査基準は**ドメイン知識**
（要点欄が空、Q&A が「これで分かりましたか？」で締まっていない、
解説本文に簡体字が混ざる…）で、汎用の LLM サービスが持つべきものではない。

選択肢:

1. 品質検査もサービスに持たせる（汎用性を捨てる）
2. 検査は呼び出し側でやり、不備なら再度キューに入れる（往復が2倍・レイテンシ増）
3. 検査基準をスキーマと一緒に渡す（表現力の設計が要る）

**机上で決めず、1経路で実地に試すこと。** ここを間違えると12か所移した後に
作り直しになる。

### 論点2: 冪等性

SQS は at-least-once。後処理が2回走ると解説が二重保存される。
`saveExplanation` などに条件付き書き込みが要る。
（参考: `goal.ts` の `saveEstimate` が
`ConditionExpression: 'attribute_exists(goalId) AND attribute_not_exists(totalSessions)'`
で同じ問題を解いている）

### 論点3: 対話系4つを非同期にするか

確認問題・採点・Q&A・目標チャットは 1〜7 秒で、画面が同期で待っている。
非同期にすると WebSocket が必須。**やる価値はあるが、最後に回すこと。**

### 論点4: WebSocket の認証

Cognito の ID トークンをソケットにどう渡すか。`$connect` で検証するか、
接続後に認証メッセージを送るか。接続テーブルの TTL と再接続の扱いも。

### 論点5: エラーの経路

LLM サービスの失敗を、どうやって後処理（＝画面に `failed` を出す側）へ届けるか。
DLQ に落ちたものを誰が拾うか。

---

## 5. 推奨する進め方

1. **設計書を書く**（イベントの形・冪等性の方式・品質検査の置き場所・WebSocket の認証）
2. **1経路だけ縦に通す** — **プログラム生成**が最適
   - 回数が少ない（1人1回）ので失敗しても被害が小さい
   - すでに非同期（starter/worker）なのでフロント変更が不要
   - 構造検査と作り直しがあるので、論点1をここで実地に決められる
3. 動いてから残り8経路を移す
4. **対話系4つ + フロントの WebSocket は最後**

### 先にやっても損しないこと（この作業と独立）

- **`usesLlm` を実態に合わせる。** 27 個のハンドラが LLM を呼んでいないのに
  API キーの読み取り権を持っている。CDK の既定が `usesLlm: true` なのが原因。
  `get-program-cost` で同型の修正を 2026-08-07 に実施済み（`readsAiUsage` を新設）。
  どの道を選んでも無駄にならない。

---

## 6. 2026-08-07 時点の状態

- dev は全経路正常。全リポジトリ push 済み、CI 成功
- 全用途の既定モデルが `gpt-5.6-luna`（環境変数 `MODEL_<用途>` で個別に戻せる）
- `durationMs` の記録が入っており、`#duration-estimates` に実績が貯まる
- stg / prod は**まだ旧 learning + goals 構成**（未移行）

### この検討で見つけて直した不具合

| | 内容 |
|---|---|
| `GET /programs/{id}` が 500 | 実施予定日の導出で goals を読むのに CDK の `readTables` に無かった |
| モデル差し替えが効いていなかった | `MODEL_SELECTION_ENV_VARS` が手書き4件で、3用途が Lambda へ渡っていなかった |
| `gpt-5.6-luna` の単価が未登録 | 全処理の原価が Opus 単価（$5/$25）で表示されていた。正しくは **$0.20 / $1.20** |
| 出題数のプロンプト矛盾 | システムプロンプトに「10問以上30問以下」が焼き込まれ、深さ light（10〜15問）でも18問が出て失敗した |

いずれも再発を検知するテストを入れてある。
