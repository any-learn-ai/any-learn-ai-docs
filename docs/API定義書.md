# Any Learn AI — API 定義書

**作成: 2026-08-09 / 状態: 実装から起こした版（dev で稼働中）**

## この文書について

⚠️ **実装が正である。**食い違いを見つけたらコードのほうを信じ、この文書を直すこと。

[基本設計書](基本設計書.md) / [詳細設計書](詳細設計書.md) と対になる。
本書はエンドポイントごとの入出力だけを扱う。

### 実装されているエンドポイント（全38 + WebSocket）

| サービス | 本数 |
|---|---|
| shared-infra | 3（`/account/providers/google`, `/dummy`, `/ws-ticket`）+ WebSocket 2ルート |
| core | 27 |
| assessments | 6 |
| media | 2 |

⚠️ **`goals` / `learning` / `programs` リポジトリの CDK にもエンドポイント定義が
あるが、デプロイされていない。**core へ統合済みで、`programs` の README には
廃止と明記されている。**読むなら core を読むこと。**

⚠️ `media` の README に `/tts` の記載があるが、**CDK にも実装にも存在しない。**

---

## 1. 共通仕様

### 1.1 認証

**REST の全エンドポイントが Cognito Authorizer 配下。**認証なしの REST は1本も無い。

```
Authorization: <Cognito ID トークン>
```

- Authorizer は `CognitoUserPoolsAuthorizer`、`identitySource` は `Authorization` ヘッダ
- 結果キャッシュ 5分
- ハンドラは `event.requestContext.authorizer.claims.sub` を `userId` として使う

WebSocket は別方式（6章）。

### 1.2 レスポンスの共通形

| 状況 | ステータス | body |
|---|---|---|
| 成功 | 200 | 各エンドポイントによる |
| 作成 | 201 | 同上 |
| **受付（非同期）** | **202** | 同上（4章） |
| 削除成功 | 204 | 空 |
| 入力が不正 | 400 | `{ error: 'invalid_request', message }` |
| 未認証 | 401 | `{ error: 'unauthorized', message }` |
| 権限なし | 403 | `{ error: 'forbidden', message }`（media のみ） |
| 見つからない | 404 | `{ error: 'not_found', message }` |
| 競合 | 409 | `{ message }`（Google 連携解除のみ） |
| サーバー障害 | 500 | `{ error: 'internal_error', message }` |

⚠️ **未処理の例外はすべて 500 に変換される**（`withErrorHandling`）。
スタックトレースは CloudWatch にのみ出る。

⚠️ **`message` は学習者が読む文言である。**画面はこれをそのまま出す。

### 1.3 CORS

```
Access-Control-Allow-Origin: *
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
```

⚠️ **API Gateway 本体を持たないサービスは、リソースごとに `addCorsPreflight` を
明示的に呼ぶ必要がある。**`defaultCorsPreflightOptions` は本体を作ったスタック内の
リソースにしか効かない。

### 1.4 リクエストの検証

⚠️ **API Gateway のリクエストバリデータは使っていない。**各ハンドラが手で検証する。
`parseBody<T>` は JSON パースに失敗すると `null` を返すだけで、型は見ない。

---

## 2. 非同期の待ち方

### 2.1 202 を受けたら

```
POST → 202 { status: 'generating', ...識別子 }
  ↓
GET を繰り返す（2〜5秒間隔）
  ↓
status が 'ready' / 'failed' になったら終了
```

⚠️ **202 の応答に生成物は入っていない。**識別子（`qaTurnIndex` /
`attemptIndex` / `chatIndex` / `questionIndex`）を持って GET を見に行く。

### 2.2 待ち画面のための2項目

生成中の GET 応答には、次の2つが入る。

| 項目 | 意味 |
|---|---|
| `startedAt` | サーバーが記録した開始時刻（ISO 8601） |
| `estimatedSeconds` | 実測から出した所要時間の目安（秒） |

⚠️ **どちらも生成中の応答にしか入らない。**完了すると `null` になる。
ポーリングの最終値だけを見ていると、待っている間ずっと目安が取れない。

### 2.3 猶予（5分）

⚠️ **`generating` のまま5分を過ぎたものは `failed` として返る。**

結果が届かなかった場合、レコードは `generating` のまま残る。そのまま返すと
画面が待ち続けるので、読むときに判断して終端させる。

このとき `message` / `error` には次が入る。

```
時間内に結果が届きませんでした。もう一度お試しください。
```

---

## 3. エンドポイント一覧

### 3.1 shared-infra

| メソッド | パス | 用途 |
|---|---|---|
| DELETE | `/account/providers/google` | Google 連携の解除 |
| GET | `/dummy` | 疎通確認（Mock 統合。`{"status":"ok"}` 固定） |
| POST | `/ws-ticket` | WebSocket 接続チケットの発行 |

### 3.2 core — 目標・計画

| メソッド | パス | 同期性 |
|---|---|---|
| POST | `/goals` | 202 |
| GET | `/goals` | 200 |
| GET | `/goals/{goalId}` | 200 |
| DELETE | `/goals/{goalId}` | 204 |
| POST | `/goals/{goalId}/chat` | 202 |
| POST | `/goals/{goalId}/plan` | 202 |
| POST | `/goals/{goalId}/plan/estimate` | 202 |

### 3.3 core — プログラム・進捗

| メソッド | パス | 同期性 |
|---|---|---|
| POST | `/programs` | 202 |
| GET | `/programs/{programId}` | 200 |
| DELETE | `/programs/{programId}` | 204 |
| GET | `/programs/{programId}/cost` | 200 |
| GET | `/progress/{programId}` | 200 |

### 3.4 core — セッション

| メソッド | パス | 同期性 |
|---|---|---|
| POST | `/sessions` | **200 / 201** |
| DELETE | `/sessions?programId=` | 200 |
| GET | `/sessions/completions?programId=` | 200 |
| GET | `/sessions/{sessionId}` | 200 |
| POST | `/sessions/{sessionId}/questions` | 202 |
| PUT | `/sessions/{sessionId}/complete` | 200 |

### 3.5 core — マイクロサイクル

| メソッド | パス | 同期性 |
|---|---|---|
| GET | `/micro-steps/{sessionId}` | 200 |
| GET | `/micro-steps/{sessionId}/next` | 200 |
| POST | `/micro-steps/{microStepId}/explanation` | **200 / 202** |
| GET | `/micro-steps/{microStepId}/explanation?sessionId=` | 200 |
| POST | `/micro-steps/{microStepId}/qa` | 202 |
| POST | `/micro-steps/{microStepId}/quiz` | **200 / 202** |
| POST | `/micro-steps/{microStepId}/evaluate` | **200 / 202** |
| POST | `/micro-steps/{microStepId}/re-explain` | 202 |
| PUT | `/micro-steps/{microStepId}/complete` | 200 |

⚠️ **パス変数は API Gateway 上すべて `{microStepId}` だが、
`GET /micro-steps/{...}` と `.../next` の2本は sessionId として扱う。**

### 3.6 assessments

| メソッド | パス | 同期性 |
|---|---|---|
| POST | `/assessments` | 202 |
| GET | `/assessments/{assessmentId}` | 200 |
| DELETE | `/assessments/{assessmentId}` | 204 |
| POST | `/assessments/{assessmentId}/answers` | **200 / 202** |
| POST | `/assessments/{assessmentId}/report` | **200 / 202** |
| GET | `/assessments/{assessmentId}/report` | 200 |

### 3.7 media

| メソッド | パス | 同期性 |
|---|---|---|
| GET | `/uploads/presigned-url` | 200 |
| DELETE | `/uploads/{key}` | 200 |

---

## 4. 200 と 202 を出し分ける5本

⚠️ **同じエンドポイントが状況で使い分ける。**クライアントは必ずステータスを見て
分岐すること。「待たなくてよいものを待たせない」ための設計である。

| エンドポイント | 200 になる条件 | 202 になる条件 |
|---|---|---|
| `POST /micro-steps/{id}/explanation` | 解説が生成済み、かつ `regenerate` でない | 未生成／`regenerate: true`／生成中（猶予内） |
| `POST /micro-steps/{id}/quiz` | 先読み済みの問題が使えて、取得に成功 | 先読み無し／前提が変わった／取り合いに負けた |
| `POST /micro-steps/{id}/evaluate` | 「分からない」を選んだ／選択式で正解一致 | 記述式／選択式で不一致 |
| `POST /assessments/{id}/answers` | 「分からない」／選択式 | 記述式 |
| `POST /assessments/{id}/report` | レポートが生成済み | 未生成／生成中 |

`POST /sessions` は **200（再開）/ 201（新規）** を出し分ける。

---

## 5. エンドポイント詳細

以下、200/400/401/404/500 の共通部分は省き、**そのエンドポイント固有のこと**だけを書く。

### 5.1 POST /goals

学習目標を作る。分析は非同期。

```ts
// リクエスト
{
  rawInput: string;                // 必須・最大 2000 文字
  inputMethod?: 'text' | 'voice';  // 'voice' 以外は 'text'
}

// 202
{ goalId, status: 'clarifying', generationStatus: 'generating' }
```

### 5.2 GET /goals

一覧。`status === 'abandoned'` は除外し、`updatedAt` 降順。

```
クエリ: limit（1〜100、既定 50）

// 200
{ goals: GoalSummary[], count }
```

`GoalSummary` の導出項目（`generationStatus` / `planStatus` / `totalSessions` /
`sessionMinutes` / `sessionsByDepth` / `startDate` / `endDate` /
`dailyLoadMinutes`）は**レコードの値を上書きして返る**。
`programStatus` はプログラムがあるときだけ入る。

### 5.3 GET /goals/{goalId}

```
// 200
{ ...Goal, ...derivedFields, programStatus?, estimatedSeconds }
```

⚠️ `estimatedSeconds` は `generationStatus === 'generating'` のときだけ値が入る。

### 5.4 POST /goals/{goalId}/chat

目標設計チャット。**返答は非同期に埋まる。**

```ts
// リクエスト
{ message: string }   // 必須・最大 1000 文字

// 202
{
  chatIndex,          // ⚠️ この番号の chatHistory を待つ
  status: 'generating',
  goal: { goalId, subject, targetLevel, context, status }
}
```

待ち方: `GET /goals/{goalId}` の `chatHistory[chatIndex].status` が
`'ready'` になれば `content` に返答が入る。`'failed'` なら `error`。

### 5.5 POST /goals/{goalId}/plan

学習計画を確定する。**確定と同時にプログラム生成が始まる。**

```ts
{
  assessmentId: string;          // 必須
  depth: 'light'|'standard'|'deep';  // 必須
  durationWeeks: number;         // 必須・整数 1〜104
  studyDays: number[];           // 必須・0(日)〜6(土)、重複除去後1件以上
  // 診断コンテキスト（任意・プログラム生成の入力になる）
  diagnosticSummary?: string;    // 先頭 4000 文字
  totalScore?: number;
  weakTopics?: string[];         // 先頭 20 件
  strongTopics?: string[];       // 同上
  estimatedLevel?: string;
}

// 202
{ goalId, planStatus: 'calculating' }
```

⚠️ **すでに `calculating` なら、ワーカーを起動せず同じ形を返す。**

⚠️ `startDate` はサーバーが当日で決める。クライアントは指定できない。

### 5.6 POST /goals/{goalId}/plan/estimate

必要セッション数の**先読み**。診断レポートを読んでいる間に走らせる。

```
// 202
{ goalId, estimateStatus: 'estimating' }   // 既存なら 'estimating' | 'ready'
```

⚠️ **保存するだけで、学習計画は確定しない。**確定は 5.5。

### 5.7 POST /programs

```ts
{ goalId: string }   // 必須。他のフィールドは無視される

// 202
{ programId, goalId, generationStatus }
```

⚠️ **すでに `generating` / `ready` のプログラムがあれば、それを返す。**
作り直しの入口として安全に残すため、`failed` のときだけ新規に作る。

### 5.8 GET /programs/{programId}

```
// 200
{
  ...Program,
  phases,              // 上書き（?? []）
  generationStatus,    // 上書き（?? 'ready'）
  generationError,     // 上書き（?? null）
  sessionSchedule,     // { [sessionId]: 'YYYY-MM-DD' }。導出不能なら {}
  startedAt,           // createdAt ?? null
  estimatedSeconds     // generating のときだけ値
}
```

### 5.9 GET /programs/{programId}/cost

```
// 200
{
  programId, calls, inputTokens, outputTokens, cacheWriteTokens, cacheReadTokens,
  usd, jpy, estimatedRate,
  byPurpose: CostBreakdownEntry[],
  byModel:   CostBreakdownEntry[],
  measuredFrom        // calls > 0 なら createdAt、0 なら null
}
```

⚠️ **`estimatedRate: true` は「単価が確かでない」ことを示す。**
呼び出しの記録に単価が入っていない古いぶんで、凍結した表から当てている。
未知のモデルは 0 円にせず**最も高い既知の単価**で見積もる。
安く見えて値付けを誤るより、高く見えるほうがよい。

### 5.10 GET /progress/{programId}

```
// 200
{
  programId, programTitle, totalWeeks, programStatus,
  generationStatus, generationError,
  completedSessionIds, currentPhaseId, currentUnitId, currentSessionId,
  lastStudiedAt, totalStudyMinutes,
  summary: {
    totalSessions, completedSessions, totalUnits, completedUnits,
    completionPercentage,
    phases: [{ phaseId, title, totalSessions, completedSessions,
               completionPercentage, isCompleted }]
  },
  completionPercentage,   // summary の再掲
  phaseProgress           // summary.phases の再掲
}
```

⚠️ **進捗は写しを持たず、プログラム構造と完了セッションから毎回導出する。**

### 5.11 POST /sessions

**200（再開）と 201（新規）を出し分ける。**

```ts
{
  programId: string;          // 必須
  unitId: string;             // 必須
  subject: string;            // 必須
  programSessionId?: string;  // あると再開判定を行う
  title?: string;             // 未指定は subject
  topicTags?: string[];
  microSteps: [{              // 必須・1〜20 件
    concept: string;          // 必須・最大 200 文字
    contentPrompt?: string;   // 最大 600 文字
    topicTags?: string[];
    estimatedMinutes?: number;         // 既定 5
    visualizationType?: 'text'|'image'|'animation'|'math';  // 列挙外は 'text'
  }]
}

// 200 / 201（同一の形）
{
  sessionId, status, currentCycleState, currentMicroStepId, totalMicroSteps,
  microSteps: [{ microStepId, order, concept, estimatedMinutes }],
  startedAt,
  resumed        // 200 なら true、201 なら false
}
```

⚠️ **再開は `programSessionId` が一致し、かつステップの概念が同じときだけ。**
プログラムを作り直すと概念が変わるので、その場合は新しいセッションになる。

### 5.12 POST /micro-steps/{microStepId}/explanation

```ts
{ sessionId: string; regenerate?: boolean }

// 200（生成済み）
{
  microStepId, concept, explanationStatus: 'ready', cached: true,
  approach, body, keyPoint,
  suggestedQuestions: [{ category, text }],   // ⚠️ internalNote は返さない
  generatedAt,
  qaTurns: [{ question, answer, status, error?, purpose? }],
  quizAttempts: [{ questionText, choices, answer, isDontKnow, verdict, score,
                   correctElements, incorrectElements, feedbackMessage,
                   hintsUsed, attemptCount, answeredAt,
                   gradingStatus, gradingError }]
}

// 202
{ microStepId, concept, explanationStatus: 'generating' }
```

⚠️ **`internalNote` は返さない。**サジェスト質問に添えた内部メモで、
回答生成の軸に使うが画面には出さない。

⚠️ **`quizAttempts` に `sampleAnswer` / `rubric` は入らない。**この形は解説と
一緒に返る＝確認問題に答える前にも届くので、混ぜると答えが先に見える。

### 5.13 GET /micro-steps/{microStepId}/explanation

**ポーリング先。**`sessionId` はクエリで**必須**。

| フィールド | 条件 |
|---|---|
| `microStepId` / `concept` / `explanationStatus` | 常時 |
| `explanationMode` | `'explanation' \| 're-explain'` があるとき |
| `explanationError` | `failed` かつ理由があるとき |
| `approach` / `body` / `keyPoint` / `suggestedQuestions` / `generatedAt` | `ready` のとき。`re-explain` なら再解説の最新分（+ `attemptCount` / `hintEnabled`） |
| `startedAt` / `estimatedSeconds` | 常時（`estimatedSeconds` は `generating` のときだけ値） |
| `quizStatus` / `quizStartedAt` | 常時 |
| `quizError` | `failed` のとき。猶予超過なら `STALE_REASON` |
| `quiz` | 出題があるとき `{ type, questionText, choices }` |
| `qaTurns` / `quizAttempts` | 常時 |

`explanationStatus` は `generating | ready | failed | not_started`。
⚠️ **猶予（5分）を過ぎた `generating` は `failed` に倒して返る。**

### 5.14 POST /micro-steps/{microStepId}/qa

```ts
{
  sessionId: string;    // 必須
  question: string;     // 必須・最大 1000 文字
  inputSource?: 'suggested' | 'self_typed' | 'voice';
  purpose?: 'hint';     // ヒント依頼
  suggestedCategory?: SuggestedQuestionCategory;
}

// 202
{ inputSource, qaTurnIndex, status: 'generating', followUpPrompt: '他に質問はありますか？' }
```

待ち方: `GET /micro-steps/{id}/explanation` の `qaTurns[qaTurnIndex].status`。

⚠️ **会話履歴はクライアントから受け取らない。**保存済みの記録から組み立てる。
送られてきた内容は検証できず、画面を再読み込みすると消えるため。

### 5.15 POST /micro-steps/{microStepId}/quiz

```ts
{
  sessionId: string;   // 必須
  quizType?: 'shortAnswer'|'multipleChoice'|'calculation'|'explanation';
}

// 200（先読みが効いた）
{
  microStepId, concept, attemptCount, hintEnabled,
  quiz: { type, questionText, choices, hintText }
}

// 202
{ microStepId, concept, attemptCount, hintEnabled, quizStatus: 'generating' }
```

⚠️ **`rubric` / `sampleAnswer` は返さない**（答えそのもの）。
`hintText` は `hintEnabled`（3回目以降）のときだけ値が入る。

⚠️ **解説が無いと 400。**`先に POST /micro-steps/{microStepId}/explanation で
解説を生成してください`

### 5.16 POST /micro-steps/{microStepId}/evaluate

```ts
{
  sessionId: string;    // 必須
  answer: string;       // 必須（dontKnow なら不要）・最大 2000 文字
  hintsUsed?: number;   // 0〜3 の整数（既定 0）
  dontKnow?: boolean;
}

// 200（その場で決まる）
{
  microStepId, gradingStatus: 'ready', attemptIndex,
  verdict, score, correctElements, incorrectElements,
  feedbackMessage, misunderstoodConcept,
  avatarState, masteryScore, isStepComplete, attemptCount,
  // 完了なら
  nextAction: 'complete',
  // 未完了なら
  nextAction: 're_explain', suggestedReExplanationApproach, hintEnabled
}

// 202
{ microStepId, gradingStatus: 'grading', attemptIndex, attemptCount }
```

⚠️ **`avatarState` / `isStepComplete` / `nextAction` /
`suggestedReExplanationApproach` は `verdict` と `score` からの決定論的な導出。**
モデルには決めさせない。

⚠️ **確認問題が無いと 400。**

### 5.17 POST /micro-steps/{microStepId}/re-explain

```ts
{
  sessionId: string;              // 必須
  userAnswer?: string;            // 最大 2000 文字
  misunderstoodConcept?: string;
}

// 202
{ microStepId, concept, explanationStatus: 'generating' }
```

⚠️ **切り口は不正解回数からサーバーが決める。**クライアントは指定しない。

### 5.18 PUT /micro-steps/{microStepId}/complete

```ts
{
  sessionId: string;            // 必須
  masteryScore?: number;        // 0.0〜1.0（既定 0）
  attemptCount?: number;        // 非負整数でなければ 1
  hintsUsed?: number;           // 同 0
  qaCount?: number;             // 同 0
  reExplanationCount?: number;  // 同 0
}

// 200
{ microStepId, status: 'completed', masteryScore,
  nextMicroStepId, isSessionComplete, sessionProgressPercent, updatedAt }
```

⚠️ **すでに完了していると 400。**`このマイクロステップはすでに完了しています`

### 5.19 POST /assessments

```ts
{
  goalId: string;        // 必須・最大 500 文字
  subject: string;       // 必須・最大 500 文字
  depth?: 'light'|'standard'|'deep';  // 既定 'deep'
  targetLevel?: string;  // 最大 500 文字
  context?: string;      // 最大 500 文字
  note?: string;         // 最大 2000 文字
}

// 202
{ assessmentId, goalId, subject, targetLevel, context,
  status: 'in_progress', generationStatus: 'generating', startedAt }
```

⚠️ **出題数は深さとテーマの広さで決まる（10〜30問）。**固定ではない。

### 5.20 GET /assessments/{assessmentId}

```
// 200
{
  assessmentId, goalId, subject, targetLevel, context, status,
  generationStatus, generationError?,
  totalScore, topicScores, answeredCount, totalQuestions,
  questionCountRationale?,     // 生成完了まで省かれる
  questions: PublicQuestion[],
  startedAt, completedAt,
  estimatedSeconds             // generating のときだけ値
}
```

⚠️ **未回答の設問からは `correctAnswer` / `explanation` を削除して返す。**
これを怠るとカンニングが成立し、診断の意味がなくなる。

⚠️ 猶予を過ぎた `grading` は `gradingStatus: 'failed'` +
`gradingError: STALE_REASON` に倒して返る。

### 5.21 POST /assessments/{assessmentId}/answers

```ts
{
  questionId: string;            // 必須
  answer: string;                // 必須（dontKnow なら不要）・最大 4000 文字
  responseTimeSeconds?: number;  // 非負の有限数のみ採用
  dontKnow?: boolean;
}

// 200（その場で決まる）
{
  assessmentId, questionId, gradingStatus: 'ready',
  isCorrect, partialScore, evaluationNote,
  correctAnswer, explanation,        // ⚠️ 回答済みになった設問のぶんだけ開示
  answeredCount, totalQuestions, status, totalScore, topicScores
}

// 202
{ assessmentId, questionId, gradingStatus: 'grading', questionIndex }
```

⚠️ **選択式の照合は前後の空白と全角空白だけを正規化する。**それ以上のゆらぎを
吸収すると、別の選択肢と一致してしまう。

⚠️ **記述式で 202 のときは総合スコアが返らない。**採点が終わってから
`GET /assessments/{assessmentId}` で取り直すこと。

### 5.22 POST /assessments/{assessmentId}/report

リクエストボディは読まない。

```
// 200（生成済み）
{ assessmentId, generationStatus: 'ready', cached: true }

// 202
{ assessmentId, generationStatus: 'generating' }
```

⚠️ **全問回答していないと 400。**`未回答の設問が残っています（n/m問 回答済み）`

### 5.23 GET /assessments/{assessmentId}/report

```
// 200（生成済み）
{ assessmentId, totalScore, topicScores, report, cached: true,
  generationStatus: 'ready' }

// 200（未生成）
{ assessmentId, totalScore, topicScores, report: null, cached: false,
  generationStatus, generationError, estimatedSeconds }
```

`report` = `{ summary, estimatedLevel, strengths[], weaknesses[],
recommendedNextSteps[], generatedAt }`

`generationStatus` は `generating | ready | failed | not_started`。

### 5.24 GET /uploads/presigned-url

```
クエリ:
  contentType  必須。image/png | image/jpeg | image/webp
                   | audio/webm | audio/mp4 | application/pdf
  extension    必須。正規化後 /^[a-z0-9]{1,5}$/

// 200
{ uploadUrl, key, expiresIn }   // expiresIn = 300
```

⚠️ **`key` はサーバーが決める**（`{userId}/{uuid}.{ext}`）。
クライアントが指定できると他人の領域へ書ける。

### 5.25 DELETE /uploads/{key}

`key` の `/` は `%2F` にエンコードして渡す。

⚠️ **403 を返す条件**: `..` を含む／`{userId}/` で始まらない／プレフィックスのみ。
他人のオブジェクトを消せないようにする。

⚠️ **404 は返さない。**存在しないキーでも S3 の DeleteObject は成功する。

### 5.26 DELETE /account/providers/google

```
// 200
{ googleLinked: false }   // 連携が無い場合も同じ

// 409
{ message: 'このアカウントは Google で作成されているため、連携を解除すると
            ログインできなくなります。' }
```

⚠️ **Google 単独で作られたアカウント（username が `google_` で始まる）は
解除させない。**解除するとログイン手段が無くなる。

---

## 6. WebSocket

### 6.1 接続の手順

```
① POST /ws-ticket        （Cognito Authorizer 配下）
     → 201 { ticket, expiresInSeconds }
② wss://{wsApiId}.execute-api.{region}.amazonaws.com/{env}?ticket={ticket}
     → $connect の Authorizer がチケットを消費して userId を確定
```

⚠️ **ID トークンをクエリ文字列に載せないこと。**クエリ文字列はアクセスログ・
CloudWatch・ブラウザ履歴に残り、ID トークンは有効期限まで再利用できる。

⚠️ **チケットは1回きり。**60秒で失効し、消費されると2本目は張れない。
再接続のたびに取り直す。

| ルート | 認可 | 応答 |
|---|---|---|
| `$connect` | CUSTOM（REQUEST Authorizer） | 200 / 401 / 500。チケットが無効なら Authorizer が Deny → **403** |
| `$disconnect` | NONE | 200 |

⚠️ **クライアントからサーバーへ送るメッセージのルートは無い。**通知は片方向。

### 6.2 届くもの

```ts
{
  kind: string;                          // replyTo と同じ（例 'qa.completed'）
  status: 'succeeded' | 'failed';
  ids: Record<string, string | number>;  // sessionId / microStepId / index など
}
```

⚠️ **生成物は入っていない。「変わった」という合図だけ。**

中身を送れば往復が1回減るが、同じデータに2つの経路ができる。届いた場合と
届かない場合で通る道が変わり、片方だけで再現する不具合になる。
**ドメインの検査を通っていない値が画面へ入る**のがいちばん危ない。

⚠️ **完了の判断は必ず GET の結果で行うこと。**合図を完了と読むと、
失敗した場合まで成功として扱うことになる。

⚠️ **識別子まで突き合わせること。**画面は複数の待ちを同時に抱えうる
（解説を読みながら先読みが走る）。`kind` だけで判断すると、関係のない合図で
読み直しが走る。

### 6.3 張れなくても動く

⚠️ **WebSocket は速めるだけの仕組みである。**

企業プロキシなどで通らない環境はある。接続に失敗したらポーリングだけで待つ。
学習には影響しないので、「接続できません」と学習者へ出さない。

`kind` に入る値の一覧は [詳細設計書 3.3](詳細設計書.md) を参照。

---

## 7. クライアント実装の注意

### 7.1 ステータスで分岐する

```ts
const result = await submitAnswer(...);
if (result.gradingStatus !== 'grading') {
  // 200。その場で結果が入っている
  show(result);
  return;
}
// 202。ポーリングへ
const graded = await pollUntilDone(() => getAssessment(id), {
  isDone:   (v) => v.questions[result.questionIndex]?.gradingStatus === 'ready',
  isFailed: (v) => v.questions[result.questionIndex]?.gradingStatus === 'failed',
  onTick:   (v) => setData(v),   // ⚠️ 待ち帯の目安を受け取る
});
```

### 7.2 失敗を3つに分ける

⚠️ **「失敗しました」に丸めないこと。**学習者が次にすべきことが違う。

| 状況 | 判定 | 勧めること |
|---|---|---|
| サーバーが `failed` を記録 | GET の `status` | もう一度作る |
| 状態を読めない（500 が続く） | GET が連続で失敗 | **作り直させない。**開き直す |
| 期限まで終わらない | クライアント側のタイムアウト | 待つ（続いている） |

⚠️ **状態を読めなかっただけで作り直しを勧めると、すでに出来上がっているものを
もう一度生成させることになる。**実際にその事故が起きた。

### 7.3 待っている間の応答を捨てない

⚠️ `estimatedSeconds` と `startedAt` は**生成中の応答にしか入らない。**
ポーリングの最終値だけを state へ入れると、待ち帯が出ないまま経過秒だけが増える。

**成功経路のテストでは見つからない。**生成が終われば画面は正しく描かれる。
