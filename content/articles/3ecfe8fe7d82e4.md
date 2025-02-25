---
title: "Flutter Cognito を利用している際に認証イベントを listen する方法"
emoji: "🍥"
type: "tech"
topics:
  - "aws"
  - "flutter"
  - "cognito"
published: true
published_at: "2024-07-03 18:20"
---

Flutter でアプリ開発をしており、IDaaS として Cognito を利用しています。
Cognito API は amplify_flutter および amplify_auth_cognito を利用しており、認証画面の構築には amplify_authenticator を利用しています。

サインイン後に、ある API を叩きたかったので、Riverpod を使ってごちゃごちゃやってたのですが、どうしてもサインイン直後の状態変化をキャッチできず放心状態になりながら公式ドキュメントを読んでいたら解決できたので備忘録として残します。

# 利用パッケージ
- amplify_authenticator v2.1.0
- amplify_auth_cognito v2.2.0
- amplify_flutter v2.2.0

# Amplify.Hub.listen をつかう
AmplifyAuth が認証フロー中に発行するイベントを listen するメソッドが amplify_flutter パッケージから提供されているのでそれをつかいます。
サインイン、サインアウト、セッション切れ、ユーザー削除のイベントに対応しており、イベントごとにコールバック関数の引数に enum が渡されるので、それを用いて条件分岐する感じで使用します。

```dart
Amplify.Hub.listen(
  HubChannel.Auth,
  (AuthHubEvent event) {
    case AuthHubEventType.signedIn:
      UserDefinedApiCall.signedIn();
      break;
    case AuthHubEventType.signedOut:
      UserDefinedApiCall.signedOut();
      break;
    case AuthHubEventType.sessionExpired:
      UserDefinedApiCall.sessionExpired();
      break;
    case AuthHubEventType.userDeleted:
      UserDefinedApiCall.userDeleted();
      break;
  }
)
```

例えばバックエンドで保管しているユーザー情報を追加で fetch したいときやログを post したいときなんかに使いやすそうです。

自分は Auth Widget を作成しているのでそこの build メソッド内で listen しています。

```dart
class AuthMain extends StatefulWidget {
  const AuthMain({
    super.key
  });

  @override 
  State<AuthMain> createState() => _State();
}

class _State extends State<AuthMain> {
  
  @override 
  Widget build(BuildContext context) {
    final subscription = Amplify.Hub.listen(
      HubChannel.Auth,
      (AuthHubEvent event) {
        switch (event.type) {
          case AuthHubEventType.signedIn:
            break;
          case AuthHubEventType.signedOut:
            break;
          case AuthHubEventType.sessionExpired:
            break;
          case AuthHubEventType.userDeleted:
            break;
        }
      }
    );
    return Authenticator(
      stringResolver: stringResolver,
      authenticatorBuilder: (
        BuildContext context,
        AuthenticatorState state
      ) {
       // 中略
      }
    );
  }
}

```

# 参考
https://docs.amplify.aws/flutter/build-a-backend/auth/connect-your-frontend/listen-to-auth-events/

# さいごに
Firebase Authentication と Flutter の組み合わせは情報も多く、自分自身でも利用経験もあったので今回やりたいこともそんなに苦労しなかった思い出があるのですが、今回は結構苦労しました。
公式ドキュメントの見づらい位置にあったわけでもないのですが見つけるのに難儀しました。英語の重要さが身に染みました。