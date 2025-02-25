---
title: "EC2 インスタンスに Tera Term から、SSH 接続をおこなう"
emoji: "🅰️"
type: "tech"
topics:
  - "aws"
  - "ec2"
  - "teraterm"
published: true
published_at: "2024-01-25 12:30"
---

久しぶりに EC2 インスタンスに Tera Term から SSH 接続をしようとしたら、いろいろと忘れてたので、備忘録として残します。

# 流れ
- タイプが「ED25519」のキーペアを用意する
- セキュリティグループを用意する
- Tera Term を用意する
- Tera Term から接続する

# (AWS 側での作業) EC2 とキーペアを用意する

好きな設定でインスタンスを起動してください。  
キーペアも同時に作成します。
今回は下記のような設定で新しくインスタンスを起動しました。

|name|value|
|-|-|
|インスタンスタイプ|t2.micro|
|AMI|Amazon Linux 2023|
|キーペア|ED25519|
|ストレージ|20GB|
|セキュリティグループ|0.0.0.0/0, ::/0 からの SSH 接続を許可|

後ほどおこなう作業の関係で、ストレージが足りない可能性があるため 20 GB にしていますが、適宜変更してください。

また、このときの注意点として、キーペアのタイプは ED25519 にします。

![](https://storage.googleapis.com/zenn-user-upload/f6e9ab850097-20240125.png)

Amazon Linux 2023 などでは SSH-RSA 署名での接続が無効化されており、代わりに rsa-sha2-256、 rsa-sha2-512 が有効化されているのですが、これらが ~v4 の Tera Term に対応していないため、接続ができないそうです。
詳細な原因については下記が詳しいです。
ちなみにこちらを読む限りだと、Tera Term のバージョンが v5 以降であれば RSA タイプでもよさそうです。

https://www.cloudbuilders.jp/articles/3200/
https://docs.aws.amazon.com/linux/al2023/ug/ssh-host-keys-disabled.html

また、インバウンドルールで SSH 接続を許可しておきましょう。

|name|value|
|-|-|
|タイプ|SSH|
|プロトコル|tcp|
|ポート範囲|22|
|ソース|0.0.0.0/0, ::/0|

![](https://storage.googleapis.com/zenn-user-upload/ad23a72c38e1-20240125.png)

# (ローカルでの作業) Tera Term から接続する

## Tera Term のインストール
下記から Tera Term を導入します。

https://teratermproject.github.io/
https://forest.watch.impress.co.jp/library/software/utf8teraterm/

## Tera Term から接続する

Tera Term を起動すると、新しい接続を行うためのダイアログが表示されます。
ホストにインスタンスのパブリック IPv4 DNS を入力します。
パブリック IPv4 DNS は、AWS の「インスタンス」一覧から確認が可能です。

![](https://storage.googleapis.com/zenn-user-upload/d1b1a8f95c91-20240125.png)

初回の接続では、下記のような警告が表示されます。
EC2 の場合、Elastic IP を割り当てている場合は known hosts に追加してもよいですが、そうでない場合、インスタンスを開始するたびに IP が変わるので、追加する必要はありません。

![](https://storage.googleapis.com/zenn-user-upload/4b9a9b8b2ab6-20240125.png)

SSH 認証のダイアログが表示されるので、ユーザー名と認証方式を入力します。
ユーザー名に関しては、使用する AMI によって変わります。
例えば Amazon Linux 2023 の場合は 「ec2-user」、Ubuntu の場合は「ubuntu」です。
認証方式は「RSA/DSA/ESDSA/ED25519鍵を使う」を選択し、秘密鍵に先ほど作成したキーペアのパスを入力します。

![](https://storage.googleapis.com/zenn-user-upload/2c2b92a0c719-20240125.png)

うまく接続ができれば、ターミナルが表示されます。

![](https://storage.googleapis.com/zenn-user-upload/af6cd1a0c508-20240125.png)

# 参考
https://teratermproject.github.io/
https://forest.watch.impress.co.jp/library/software/utf8teraterm/
https://www.cloudbuilders.jp/articles/3200/
https://docs.aws.amazon.com/linux/al2023/ug/ssh-host-keys-disabled.html

