---
title: "AWS CLI から Cognito で作成したユーザーにサインインしてトークンを取得する"
emoji: "🅰️"
type: "tech"
topics:
  - "aws"
  - "cognito"
  - "awscli"
published: true
published_at: "2024-02-01 20:06"
---

Rails API モードで REST API を構築しています。
認証基盤としては Cognito を使用しております。
一部の API でトークンの検証を行いたかったので、コンソールからユーザーを作成し、
AWS CLI からサインインとパスワード認証を行い、トークンを取得しました。

# 準備するもの
- Cognito ユーザープール
- AWS CLI
- 適当な IAM ロール
    - **AmazonCognitoPowerUser** ポリシーをアタッチしています

# 流れ
- ユーザー作成
- サインイン
- 確認

# ユーザー作成(サインアップ)
対象のユーザープールからユーザーを作成します。
![](https://storage.googleapis.com/zenn-user-upload/5bd5a9c9cf11-20240201.png)

招待メッセージは「**送信しない**」、また、「**E メールアドレスを検証済みとしてマークする**」にチェックを入れてください。
メールアドレスとパスワードは任意のもので大丈夫です。

ちなみにユーザー作成も CLI から可能なのですが、テストユーザーしか作らないので、調べてないです。
参考にした記事ではサインアップからやっていました。
https://dev.classmethod.jp/articles/sign-up-and-sign-in-by-cognito-with-awscli/

# サインイン
下記コマンドを実行して、サインインを試みます。
`user-pool-id` はユーザープールの概要から、`client-id` は「アプリケーションの統合->アプリケーションクライアントのリスト」から確認できます。

```bash
aws cognito-idp admin-initiate-auth \
--user-pool-id region_xxxxxxx \
--client-id zzzzzzzzzzzzzzzzzzzzzzzzz \
--auth-flow ADMIN_USER_PASSWORD_AUTH \
--auth-parameters "USERNAME=<設定したメールアドレス>,PASSWORD=<設定したパスワード>"
```

パスワードの変更を要求された場合は、以下のようなレスポンスが返ってきます。

```json
{
    "ChallengeName": "NEW_PASSWORD_REQUIRED",
    "Session": <とても長い文字列>,
    "ChallengeParameters": {
        "USER_ID_FOR_SRP": "xxxxxxxxxx-xxxxxxxx-xxxxxxxxx-xxxxx",
        "requiredAttributes": "[]",
        "userAttributes": "{\"email_verified\":\"true\",\"email\":\"mailaddress\"}"
    }
}
```

パスワードを変更します。
session には、↑でえられた "Session" の値を入れて下さい。有効期限が設定されているので、期限切れになったら、最初のサインインからやり直してください。

```bash
aws cognito-idp admin-respond-to-auth-challenge \
--user-pool-id region_xxxxxxx \
--client-id zzzzzzzzzzzzzzzzzzzzzzzzz \
--challenge-name NEW_PASSWORD_REQUIRED \
--challenge-responses NEW_PASSWORD='<設定したメールアドレス>',USERNAME=username or mailaddress etc.\
--session  "<とても長い文字列>"
```

# 確認
うまくいくと以下のようなレスポンスが得られます。

```json
{
    "ChallengeParameters": {},
    "AuthenticationResult": {
        "AccessToken": "とても.ながい.もじれつ",
        "ExpiresIn": 3600,
        "TokenType": "Bearer",
        "RefreshToken": "めっちゃ.ながい.もじれつ",
        "IdToken": "とても.ながい.もじれつ",
    }
}
```

# さいごに
IAM ロールのポリシーとセッション切れに注意します。

# 参考
https://dev.classmethod.jp/articles/sign-up-and-sign-in-by-cognito-with-awscli/
https://dev.classmethod.jp/articles/change-cognito-user-force_change_passwore-to-confirmed/
https://docs.aws.amazon.com/cli/latest/reference/cognito-idp/