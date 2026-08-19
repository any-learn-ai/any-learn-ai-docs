# Cognito ユーザープール作り直し手順

⚠️ **この文書は、実行する日に上から順に読む手順書である。**
設計の説明ではない。判断の根拠は各手順の中に置いた。

---

## 0. なぜ作り直すのか

いまのプールは `email` を **必須（`Required: true`）** で作ってある。
⚠️ **そのせいで Google の属性マッピングから `email` を外せない。**

```
The attribute mapping is missing required attributes [email]
```

外せないと、⚠️ **Google でサインインするたびに Cognito が利用者の
メールアドレスを Google 側の値で上書きする。**エラーは出ない。

### ⚠️ この3つが、いまはできない

| やりたいこと | いまの状態 |
|---|---|
| メールアドレスの変更 | ⚠️ Google 連携中は**変えても黙って戻る** |
| 別アドレスの Google を後から連携 | ⚠️ **別アカウントができる** |
| パスワードの追加 | ✅ できる（案C で実装済み） |

### 実測（2026-08-19・すべて使い捨てプールで確認）

| プール | ログイン方式 | `email` | マッピングを外せるか |
|---|---|---|---|
| A | `UsernameAttributes` | **必須** | ❌ |
| B | `AliasAttributes` | 任意 | ✅ |
| **C** | **`UsernameAttributes`（いまと同じ）** | **任意** | ✅ |
| D | `AliasAttributes` | **必須** | ❌ |

⚠️ **効いているのは `Required` だけ。**ログイン方式は変えなくてよい。

そして本物の Google 認証で、`PreSignUp` に届くものも確認した。

```json
{
  "triggerSource": "PreSignUp_ExternalProvider",
  "userName": "Google_117837252700134746143",
  "userAttributes": {
    "email_verified": "true",
    "custom:googleEmail": "（Google のアドレス）"
  }
}
```

⚠️ **`email` は入らず、`custom:googleEmail` に入る。**紐づけに必要な
情報は失われない。

### ⚠️ なぜ作り直しなのか

`Required` は**既存プールでは変更できない**。
`update-user-pool` にスキーマを変える口が無く、追加以外の変更は

```
Existing schema attributes cannot be modified or deleted
```

で拒否される。⚠️ **プールを作り直す以外に道が無い。**

---

## 1. ⚠️ いま実行すべき理由

**利用者数（2026-08-19 時点）**

| 環境 | Cognito ユーザー | 契約レコード |
|---|---|---|
| dev | 6（全員が開発用） | 5 |
| stg | **0** | 0 |
| prod | **0** | 0 |

⚠️ **本番の利用者は1人もいない。**この作業の費用は、これ以降どの日を
選んでも今日より高くなる。⚠️ **1人でも課金利用者が付いた瞬間、
「全員が再登録」という説明のつかない事態になる。**

---

## 2. 失われるもの

### ⚠️ 全ユーザーの sub が変わる

`userId` は Cognito の `sub` をそのまま使っている。**写しはどこにも
無い**（users テーブルは意図的に廃止した）。⚠️ **旧 sub → 新 sub の
対応表を作る手掛かりがシステム内に存在しない。**

sub を主キーにしているもの:

| リポジトリ | テーブル / 資源 | キー |
|---|---|---|
| shared-infra | `entitlements` | `userId` |
| shared-infra | `usage-counters` | `userId` / `period` |
| shared-infra | `user-preferences` | `userId` |
| shared-infra | `ws-connections` GSI | `userId` |
| core | `sessions` | `userId` / `sessionId` |
| core | `sessions` GSI | ⚠️ `userId#programId` の**合成キー** |
| core | `micro-step-progress` | `userId` / `microStepId` |
| core | `usage-counters` | `userId` / `period` |
| core | `goals` | `userId` / `goalId` |
| core | `programs` | `userId` / `programId` |
| assessments | `assessments` | `userId` / `assessmentId` |
| assessments | `usage-counters` | `userId` / `period` |
| llm | `ai-usage` | ⚠️ `userId#scopeId` の**合成キー** |
| media | S3 のキー | ⚠️ `{userId}/...` の**接頭辞** |

⚠️ **合成キーと S3 の接頭辞は、単純な置換では移せない。**

### ⚠️ パスワードは移行できない

Cognito はパスワードのハッシュを取り出せない。⚠️ **全員に再登録か
パスワード再設定を案内するしかない。**

### ⚠️ Stripe の購読に旧 sub が焼き込まれている

- `billing-session.ts` — `client_reference_id: userId` / `metadata.userId`
- `stripe-webhook.ts` — `metadata.userId` を読んで契約を更新

⚠️ **dev に購読が1件ある**（`professional` / `cus_V4aUyS9x3MybK8`）。
prod は0件。⚠️ **prod に購読が出てからでは、Stripe 側の metadata を
手で書き換える作業が発生する。**

---

## 3. 事前準備

### 3.1 ⚠️ ドメイン接頭辞の衝突（stg / prod のみ）

ホスト UI の接頭辞は `any-learn-ai-{env}` で固定（`config.ts`）。
⚠️ **AWS 全体で一意**である必要がある。

そして削除方針は:

| 環境 | `isProduction` | `removalPolicy` |
|---|---|---|
| dev | false | `DESTROY` |
| **stg** | **true** | ⚠️ **`RETAIN`** |
| **prod** | **true** | ⚠️ **`RETAIN`** |

⚠️ **stg も RETAIN である。**旧プールが残ったまま同じ接頭辞で新しい
ドメインを作ろうとして、⚠️ **デプロイが失敗する。**

**対処: stg / prod では、新スタックを流す前に旧プールのドメインを消す。**

```bash
aws cognito-idp delete-user-pool-domain \
  --user-pool-id <旧プールID> --domain any-learn-ai-<env>
```

⚠️ **接頭辞を変える案は採らない。**変えると Google Cloud Console の
承認済みリダイレクト URI を手で直すことになり、⚠️ **手順が1つ増える
だけでなく、直し忘れると Google ログインが丸ごと止まる。**

### 3.2 sub とメールの対応を退避する

⚠️ **消す前に取ること。**消してからでは取れない。

```bash
aws cognito-idp list-users --user-pool-id <旧プールID> \
  --query 'Users[].{u:Username,a:Attributes}' --output json > old-users.json
```

⚠️ **dev では使わない**（データを捨てる前提）。⚠️ **利用者が居る環境で
実行する日には、これを飛ばさないこと。**

### 3.3 Google Cloud Console

⚠️ **接頭辞を変えないなら、変更は不要。**リダイレクト URI は
`https://any-learn-ai-{env}.auth.ap-northeast-1.amazoncognito.com/oauth2/idpresponse`
のままである。

---

## 4. 手順

### 4.1 コードを変える（shared-infra）

**① プールの論理 ID を上げる**

```
lib/shared-infra-stack.ts
  new cognito.UserPool(this, 'UserPoolV2', ...)
    → 'UserPoolV3'
```

⚠️ **論理 ID を変えないと作り直されない。**属性の変更は黙って無視される
（`V2` を付けた前例が、まさにこの理由）。

**② `email` を任意にする**

```ts
standardAttributes: {
  email: { required: false, mutable: true },   // ⚠️ required を false へ
}
```

⚠️ **`mutable: true` は絶対に下げない。**外部 IdP の属性同期が失敗して
認証ごと落ちる（過去にプールを作り直した原因がこれ）。

**③ カスタム属性を足す**

```ts
customAttributes: {
  googleEmail: new cognito.StringAttribute({ mutable: true }),
}
```

**④ Google の属性マッピングから `email` を外す**

```ts
attributeMapping: {
  custom: {
    'custom:googleEmail': cognito.ProviderAttribute.GOOGLE_EMAIL,
    email_verified: cognito.ProviderAttribute.other('email_verified'),
  },
}
```

⚠️ **`nickname` も外す。**同じ理由で、利用者が変えた表示名を上書きする。

**⑤ `PreSignUp` の読み先を変える**

```
src/pre-signup-link.ts
  attrs.email  →  attrs['custom:googleEmail']
```

⚠️ **ここを直し忘れると、Google 登録が全部「新規アカウント」になる。**
既にアカウントを持っている人に2つ目ができ、学習データが分かれる。
⚠️ **落ちないので気づけない。**

**⑥ 番人テストを直す**

- `test/shared-infra-stack.test.ts` — `email` が `Required: false` であること
- `test/pre-signup-link.test.ts` — `custom:googleEmail` から読むこと

### 4.2 デプロイの順序

⚠️ **順序を守ること。**Authorizer の ID が変わるため、追随しないスタックは
**全 API が 401 になる。**

```
① shared-infra          ← 新プール・新 Authorizer・SSM 更新
② core / assessments / media / llm   ← Authorizer を SSM から取り直す
③ frontend              ← ⚠️ 再ビルドしないと旧プール ID が焼き込まれたまま
```

⚠️ **② を飛ばすと、旧 Authorizer ID を指したまま動く。**
⚠️ **③ は静的書き出しなので、再デプロイではなく再ビルドが要る。**

⚠️ **手元から `cdk deploy` しないこと。**CI が渡す文脈
（`googleOAuth` / `MAIL_FROM_ADDRESS` / Stripe の変数）が無いため、
⚠️ **Google 連携・SES・OAuth 設定・Stripe の配線が「不要」と判断されて
消える。**必ず GitHub Actions 経由で流す。

### 4.3 後始末

- ⚠️ **旧プールを消す**（stg / prod は `RETAIN` なので手で消す）
- `tools/backfill-entitlements.mjs` に新しい `USER_POOL_ID` を渡して実行
  ⚠️ 新プールでは全員が契約レコード無し。判定は fail-open なので
  **落ちないまま上限だけ効かない**状態になる
- ローカルの `cdk-outputs.json`（git 管理外）に旧 ID が残る
- ⚠️ `userPoolArn` の SSM パラメータは**どこからも読まれていない**。
  ついでに消す候補

---

## 5. 確認項目

⚠️ **上から順に、1つずつ確かめる。**まとめて確認しない。

### 5.1 プールの形

```bash
aws cognito-idp describe-user-pool --user-pool-id <新> \
  --query 'UserPool.{Schema:SchemaAttributes[?Name==`email`],Update:UserAttributeUpdateSettings}'
```

- [ ] `email` が `Required: false`
- [ ] `Mutable: true`
- [ ] `AttributesRequireVerificationBeforeUpdate: ["email"]`
      ⚠️ 無いと、アドレスを打ち間違えた人が**その場で締め出される**

### 5.2 Google の設定

```bash
aws cognito-idp describe-identity-provider --user-pool-id <新> --provider-name Google \
  --query 'IdentityProvider.AttributeMapping'
```

- [ ] `email` が**入っていない**
- [ ] `custom:googleEmail` が入っている

### 5.3 経路

- [ ] メール＋パスワードで新規登録できる
- [ ] Google で新規登録できる（⚠️ ユーザー名が **UUID**。`Google_` ではない）
- [ ] 契約レコード（Free）が **sub** で作られている
- [ ] ⚠️ 英語の招待メールが飛んでいない
- [ ] パスワード再設定（「お忘れですか」）が動く
- [ ] Google だけの人が、再設定からパスワードを設定できる
- [ ] ⚠️ **アドレスを変更し、Google でログインし直しても戻らない**
      ← **これが今回の目的そのもの**
- [ ] 学習の一連（目標 → 診断 → プログラム → セッション）が通る
- [ ] WebSocket が繋がる（⚠️ プール非依存だが、一応）

---

## 6. ⚠️ 利用者が居る状態で実行する場合（将来のため）

⚠️ **上の手順だけでは足りない。**以下が追加で要る。

1. **告知** — 全員が強制ログアウトされ、パスワードは移行できない
2. **sub の対応表** — 3.2 の退避を**必ず**行う。新プールでの登録後、
   メールアドレスで突き合わせて旧 sub を引く
3. **データの移行** — 14 か所（2章の表）。⚠️ **合成キー2つと S3 の
   接頭辞は単純置換では移せない**
4. **Stripe** — 既存購読の `metadata.userId` を新 sub へ書き換える
5. **停止時間** — 移行中に書き込みが走ると取りこぼす。⚠️ **止めるか、
   二重書きの仕組みが要る**

⚠️ **これを避けるために、利用者が0人のうちに実行する。**

---

## 7. 引き返す

⚠️ **プールを作った後は引き返せない。**新プールで登録した人の
アカウントは、旧プールに存在しない。

引き返せるのは **4.1 のコード変更まで**。デプロイを始めたら前に進む。

⚠️ **失敗しうる箇所と、そのときの状態:**

| どこで失敗 | 状態 |
|---|---|
| shared-infra のデプロイ | ⚠️ ロールバックが**属性追加で止まる**ことがある。`continue-update-rollback --resources-to-skip` で復旧（実例あり） |
| ドメインの衝突 | 新プールはできるがドメインが無い。旧ドメインを消して再実行 |
| core / assessments の追随漏れ | ⚠️ **全 API が 401。**該当スタックを流し直す |
| frontend の再ビルド漏れ | ⚠️ **ログインできない。**再ビルドする |

---

## 8. この文書を書くまでに間違えたこと

⚠️ **同じ間違いを繰り返さないために残す。**

1. **「マッピングは外せない」と結論した** → ⚠️ `email` が任意なら外せた。
   1つのプールで試して一般化した
2. **「プールごと作り直しで sub が全部変わる」と重く見積もった** →
   ⚠️ sub が変わるのは事実だが、**変える設定は1点だけ**だった
3. ⚠️ **`cdk diff` が「置き換えではない」と言ったので安全だと判断した** →
   **「置き換えでない」と「取り消せる」は別のこと。**属性の追加は
   不可逆で、同時に入れた変更が失敗すると**戻せないものだけが残る。**
   ⚠️ **不可逆な変更は単独でデプロイすること**
4. **実測せずに Cognito の挙動を推論した（3回外した）** →
   ⚠️ **使い捨てプールは数分で作れる。**推論より先に測る
