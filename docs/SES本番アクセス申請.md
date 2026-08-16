# Amazon SES 本番アクセス申請（サンドボックス解除）

*最終更新: 2026-08-16*

⚠️ **却下されても理由は教えてもらえない。**返ってくるのは
「セキュリティ上の理由で具体的な基準は開示できません」だけである。
だから、出す前にこちらで潰せるものは全部潰しておく。

---

## 0. まず知っておくこと

### ⚠️ フォームから「ユースケースの説明」欄が消えた

以前のフォームにあった自由記述欄は削除され、API の
`UseCaseDescription` パラメータも
`This parameter has been deprecated.` と明記されている。

**では説明はどこへ書くのか。**送信すると AWS が自動でサポートケースを
開く。⚠️ **そのケースに、聞かれる前に自分から返信する。**
承認された事例はどれもこの形だった。

### ⚠️ 一度出したら、審査中は編集できない

「Once you submit a review of your account details, you can't edit your
details until the review is complete.」

`put-account-details` を再実行すると `ConflictException` になる。
一度きりの勝負なので、下の準備がすべて済んでから出すこと。

### ⚠️ CLI ではなくコンソールから出す

`aws sesv2 put-account-details` は使わない。理由は3つ。

- Acknowledgement のチェックが記録されない
- コンソール側の事前チェックが走らない
- `--use-case-description` が deprecated で読まれる保証がない

### 本文は英語で書く

日本語で出して返事が来ず、英語で出し直して3日で通った報告がある。
希望言語を日本語にしておけば、通知だけ日本語で届く。

---

## 1. 出す前に揃えるもの

⚠️ **申請文に書くことは、全部**本当に**しておくこと。**
書いてあるのにやっていないと、そこで落ちる。

| | 確かめ方 |
|---|---|
| DKIM が SUCCESS | `aws sesv2 get-email-identity --email-identity anylearn.jp --query DkimAttributes.Status` |
| カスタム MAIL FROM が SUCCESS | 同上 `--query MailFromAttributes` |
| SPF が1本だけ | `dig TXT anylearn.jp` |
| DMARC の報告先が実在する | `dig TXT _dmarc.anylearn.jp` |
| 抑制リストが BOUNCE / COMPLAINT で有効 | `aws sesv2 get-account --query SuppressionAttributes` |
| 設定セットが CloudWatch へ記録する | `aws sesv2 get-configuration-set-event-destinations --configuration-set-name any-learn-ai-prod-mail` |
| 評判のアラームがある | `aws cloudwatch describe-alarms --alarm-name-prefix any-learn-ai` |
| サイトが生きている | `curl -o /dev/null -w '%{http_code}' https://anylearn.jp/` |
| 規約・プライバシーポリシーが読める | `/legal/terms/` `/legal/privacy/` `/legal/commerce/` |
| シミュレータで疎通を確認した | 下記 |

### ⚠️ シミュレータでの疎通確認

設定セットは**作っただけでは1件も記録しない。**送信時に指定されて
初めて効く。Cognito へ渡す配線が生きているかは、送ってみないと
分からない。

```bash
for kind in success bounce complaint; do
  aws sesv2 send-email --region ap-northeast-1 \
    --from-email-address "AnyLearn <noreply@anylearn.jp>" \
    --destination "ToAddresses=${kind}@simulator.amazonses.com" \
    --configuration-set-name any-learn-ai-prod-mail \
    --content "Simple={Subject={Data=test,Charset=UTF-8},Body={Text={Data=test,Charset=UTF-8}}}"
done
```

数分後、メトリクスに出ることを確認する。

```bash
aws cloudwatch get-metric-statistics --region ap-northeast-1 \
  --namespace AWS/SES --metric-name Bounce \
  --start-time "$(date -u -d '30 minutes ago' +%Y-%m-%dT%H:%M:%SZ)" \
  --end-time "$(date -u +%Y-%m-%dT%H:%M:%SZ)" --period 3600 --statistics Sum \
  --dimensions Name=ses:configuration-set,Value=any-learn-ai-prod-mail
```

⚠️ シミュレータ宛は抑制リストに載らないし、バウンス率にも数えられない。
AWS が意図的に除外している。安心して何度でも試せる。

---

## 2. コンソールのフォーム

```
SES → Account dashboard → View Get set up page → Request production access
```

| 項目 | 値 |
|---|---|
| メールタイプ | Transactional |
| Website URL | `https://anylearn.jp` |
| 追加の連絡先 | 受け取れるアドレス（最大4件） |
| 希望言語 | 日本語 |
| Acknowledgement | チェック |

---

## 3. サポートケースへ貼る本文

⚠️ **送信直後に、聞かれるのを待たずに貼る。**

```
Dear Amazon Web Services Team,

We operate AnyLearn (https://anylearn.jp), an AI-assisted personalized
learning service for individual learners in Japan. We plan to use Amazon SES
in ap-northeast-1 for transactional email only. No marketing, promotional,
bulk, or cold-outreach email will ever be sent from this account.

1. Email Purpose
   - Account sign-up verification codes (Amazon Cognito)
   - Password reset codes (Amazon Cognito)
   These are the only messages we send. Every one is 1:1 and is triggered by
   an explicit action the recipient performed on our own website.

2. Sending Frequency
   Approximately 10-50 emails per day, roughly 300-1,500 per month at launch.
   Peak throughput is well under 1 message per second.

3. Recipient Management
   Email addresses are collected only from our own sign-up form, entered by
   the person who owns the address. We have never purchased, rented, scraped,
   or imported any mailing list, and we do not send to addresses we did not
   receive directly from the recipient.

4. Bounce and Complaint Handling
   - An SES configuration set (any-learn-ai-prod-mail) publishes Send,
     Delivery, Bounce, Complaint, Reject and RenderingFailure events to
     Amazon CloudWatch. The Cognito user pool is configured to send through
     this configuration set.
   - Account-level suppression is enabled for BOUNCE and COMPLAINT, so a hard
     bounce or a complaint suppresses that address automatically and it is
     never retried.
   - CloudWatch alarms fire at a 2.5% bounce rate and a 0.05% complaint rate,
     which is half of the AWS review thresholds (5% and 0.1%), so we are
     alerted well before our sending would be placed under review.
   - On a complaint we stop all mail to that recipient immediately and send
     no further message of any kind, including no confirmation message.
   - We have verified this pipeline end to end using the Amazon SES mailbox
     simulator (success@, bounce@ and complaint@simulator.amazonses.com) and
     confirmed that the Bounce and Complaint events are recorded.

5. Opt-out
   Because every message is a 1:1 system notification with no promotional
   content, a marketing unsubscribe is not applicable. Recipients stop all
   email by deleting their account, or by contacting support@anylearn.jp;
   such addresses are added to the SES account-level suppression list
   immediately.

6. Sample Email
   From:    AnyLearn <noreply@anylearn.jp>
   Subject: AnyLearn 確認コード   (AnyLearn verification code)
   Body:
     AnyLearn をご利用いただきありがとうございます。

     確認コード: 123456

     画面にこのコードを入力してください。
     お心当たりが無い場合は、このメールを破棄してください。

   (Thank you for using AnyLearn. Verification code: 123456. Please enter
    this code on the screen. If you did not request this, please discard
    this email.)

7. Domain Authentication
   - Domain anylearn.jp is verified in SES; Easy DKIM status is SUCCESS.
   - SPF is published as a single record and includes amazonses.com.
   - A custom MAIL FROM domain (mail.anylearn.jp) is configured and verified.
   - DMARC is published with p=none and rua=mailto:dmarc@anylearn.jp, which
     is a real, monitored mailbox.

8. Policies published on our site
   - Terms of Service:  https://anylearn.jp/legal/terms/
   - Privacy Policy:    https://anylearn.jp/legal/privacy/
   - Commercial disclosure (Japanese Specified Commercial Transactions Act):
     https://anylearn.jp/legal/commerce/

We will use Amazon SES in accordance with the AWS Acceptable Use Policy.
Thank you for your consideration.

AnyLearn
https://anylearn.jp
```

---

## 4. 承認された後にやること

⚠️ **承認を確認するまで、絶対に切り替えないこと。**
サンドボックスでは検証済みのアドレス宛にしか送れない。切り替えた
瞬間に、新規登録した人**全員**へ確認コードが届かなくなる。
Cognito の既定送信のほうがまだましである。

```bash
aws sesv2 get-account --region ap-northeast-1 --query ProductionAccessEnabled
# true になってから
```

```bash
for e in dev stg prod; do
  gh variable set MAIL_FROM_ADDRESS -R any-learn-ai/any-learn-ai-shared-infra \
    --env "$e" --body noreply@anylearn.jp
done
```

そのうえで shared-infra を3環境へデプロイする。

---

## 5. 却下されたときにやること

⚠️ **同じ文章で出し直さない。**新しい材料を足す。

報告のある「効いた材料」は2つ。

1. **同一組織の他アカウントでの承認実績**（サポートケース ID を書く）
2. **`ses:Recipients` 条件による送信先の制限**
   「信じてください」を「そもそも悪用できません」に変えられる

⚠️ 却下後はコンソールのボタンが押せなくなり、
`put-account-details` も `ConflictException` を返すことがある。
その場合は Service Quotas から出す。

```bash
aws service-quotas request-service-quota-increase \
  --service-code ses --quota-code <該当のコード> --desired-value 50000
```

⚠️ コード番号は Service Quotas のコンソールで確認すること。
出典が単一の記事しか無く、裏が取れていない。

---

## 6. 送信上限について

**本番アクセスと送信上限の引き上げは別の手続きである。**

- 本番アクセス … SES のコンソール（この文書）
- 送信上限 … Service Quotas のコンソール

⚠️ **同時に大きな上限を求めない。**新しいアカウントで不相応な数を
求めるのは、自動却下がいちばん反応する形である。まず本番アクセスを
取り、実績を積めば AWS が自動で引き上げる。

サンドボックス中の上限は 1日200通・毎秒1通。
解除後の既定はおおむね 1日50,000通。

⚠️ **いまの Cognito 既定送信は1日50通である。**prod で新規登録が
1日50人を超えると、そこで頭打ちになる。
