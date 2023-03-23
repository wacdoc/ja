# 独自の SMTP メール送信サーバーを構築する

## 前文

SMTP は、次のようなサービスをクラウド ベンダーから直接購入できます。

* [Amazon SES SMTP](https://docs.aws.amazon.com/ses/latest/dg/send-email-smtp.html)
* [アリ クラウド メール プッシュ](https://www.alibabacloud.com/help/directmail/latest/three-mail-sending-methods)

独自のメール サーバーを構築することもできます - 無制限の送信、全体的なコストの削減。

以下に、独自のメールサーバーを構築する方法を順を追って示します。

## サーバーの選択

自己ホスト型 SMTP サーバーには、ポート 25、456、および 587 が開いているパブリック IP が必要です。

一般的に使われているパブリック クラウドはデフォルトでこれらのポートをブロックしており、ワーク オーダーを発行することで開くことができる場合もありますが、非常に面倒です。

これらのポートが開いていて、リバース ドメイン名の設定をサポートしているホストから購入することをお勧めします。

ここでは、[コンタボを](https://contabo.com)お勧めします。

Contabo は、ドイツのミュンヘンに本拠を置くホスティング プロバイダーで、2003 年に設立され、非常に競争力のある価格を提供しています。

購入通貨にユーロを選択すると、価格が安くなります（メモリ8GB、CPU4基のサーバーで年間約529元、初期導入費用は1年間無料）。

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/UoAQkwY.webp)

ご注文の際は`prefer AMD` 。AMD CPU を搭載したサーバーの方がパフォーマンスが優れています。

以下では、Contabo の VPS を例として、独自のメール サーバーを構築する方法を説明します。

## Ubuntu システム構成

ここでのオペレーティング システムは Ubuntu 22.04 です。

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/smpIu1F.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/m7Mwjwr.webp)

ssh 上のサーバーに`Welcome to TinyCore 13!`表示された場合 (下図のように)、システムがまだインストールされていないことを意味します. ssh を切断し、数分待ってから再度ログインしてください.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/-qKACz9.webp)

`Welcome to Ubuntu 22.04.1 LTS`が表示されたら、初期化が完了し、次の手順に進むことができます。

### [オプション] 開発環境を初期化する

この手順はオプションです。

便宜上、ubuntu ソフトウェアのインストールとシステム構成を[github.com/wactax/ops.os/tree/main/ubuntu](https://github.com/wactax/ops.os/tree/main/ubuntu)に置きました。

次のコマンドを実行して、ワンクリックでインストールします。

```
bash <(curl -s https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

中国のユーザーは、代わりに次のコマンドを使用してください。言語、タイム ゾーンなどが自動的に設定されます。

```
CN=1 bash <(curl -s https://ghproxy.com/https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

### Contabo は IPV6 を有効にします

SMTP が IPV6 アドレスの電子メールも送信できるように、IPV6 を有効にします。

`/etc/sysctl.conf`を編集

次の行を変更または追加します

```
net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
net.ipv6.conf.lo.disable_ipv6 = 0
```

[contabo チュートリアルのフォローアップ: サーバーへの IPv6 接続の追加](https://contabo.com/blog/adding-ipv6-connectivity-to-your-server/)

`/etc/netplan/01-netcfg.yaml`を編集し、下の図に示すようにいくつかの行を追加します (Contabo VPS の既定の構成ファイルには既にこれらの行が含まれているため、コメントを解除するだけです)。

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/5MEi41I.webp)

次に`netplan apply`て、変更された構成を有効にします。

構成が成功したら、 `curl 6.ipw.cn`使用して、外部ネットワークの ipv6 アドレスを表示できます。

## 構成リポジトリ ops のクローンを作成します

```
git clone https://github.com/wactax/ops.soft.git
```

## ドメイン名の無料 SSL 証明書を生成する

メールの送信には、暗号化と署名のための SSL 証明書が必要です。

[acme.sh](https://github.com/acmesh-official/acme.sh)を使用して証明書を生成します。

acme.sh は、オープン ソースの自動証明書署名ツールです。

構成ウェアハウス ops.soft に入り、 `./ssl.sh`を実行すると、**上位ディレクトリ**に`conf`フォルダーが作成されます。

[acme.sh dnsapi](https://github.com/acmesh-official/acme.sh/wiki/dnsapi)から DNS プロバイダーを見つけ、 `conf/conf.sh`を編集します。

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Qjq1C1i.webp)

次に、 `./ssl.sh 123.com`を実行して、ドメイン名の`123.com`および`*.123.com`証明書を生成します。

最初の実行では、自動的に[acme.sh](https://github.com/acmesh-official/acme.sh)がインストールされ、自動更新用のスケジュールされたタスクが追加されます。 `crontab -l`見ると、以下のような行があります。

```
52 0 * * * "/mnt/www/.acme.sh"/acme.sh --cron --home "/mnt/www/.acme.sh" > /dev/null
```

生成された証明書のパスは、 `/mnt/www/.acme.sh/123.com_ecc。`

証明書の更新は`conf/reload/123.com.sh`スクリプトを呼び出し、このスクリプトを編集します。nginx `nginx -s reload`などのコマンドを追加して、関連するアプリケーションの証明書キャッシュを更新できます。

## chasquid で SMTP サーバーを構築する

[chasquid は](https://github.com/albertito/chasquid)Go 言語で書かれたオープン ソースの SMTP サーバーです。

Postfix や Sendmail などの古いメール サーバー プログラムの代替として、chasquid はよりシンプルで使いやすく、二次開発も容易です。

`./chasquid/init.sh 123.com`ワンクリックで自動的にインストールされます (123.com を送信ドメイン名に置き換えます)。

## 電子メール署名 DKIM の構成

DKIM は、電子メールの署名を送信して、手紙がスパムとして扱われるのを防ぐために使用されます。

コマンドが正常に実行されると、DKIM レコードを設定するように求められます (以下を参照)。

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/LJWGsmI.webp)

TXT レコードを DNS に追加するだけです (以下を参照)。

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/0szKWqV.webp)

## サービスのステータスとログを表示

 `systemctl status chasquid`サービスのステータスを表示します。

正常動作時の状態は下図の通り

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/CAwyY4E.webp)

 `grep chasquid /var/log/syslog`または`journalctl -xeu chasquid`エラー ログを表示できます。

## 逆ドメイン名の構成

逆ドメイン名は、IP アドレスを対応するドメイン名に解決できるようにするためのものです。

逆引きドメイン名を設定すると、メールがスパムとして識別されるのを防ぐことができます。

メールが受信されると、受信サーバーは送信サーバーの IP アドレスに対して逆ドメイン名分析を実行し、送信サーバーが有効な逆ドメイン名を持っているかどうかを確認します。

送信側サーバーに逆引きドメイン名がない場合、または逆引きドメイン名が送信側サーバーの IP アドレスと一致しない場合、受信側サーバーは電子メールをスパムとして認識したり、拒否したりする可能性があります。

[https://my.contabo.com/rdns](https://my.contabo.com/rdns)にアクセスし、以下のように構成します

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/IIPdBk_.webp)

逆引きドメイン名を設定した後、ドメイン名 ipv4 および ipv6 のサーバーへの順方向解決を忘れずに構成してください。

## chasquid.conf のホスト名を編集します

`conf/chasquid/chasquid.conf`を逆ドメイン名の値に変更します。

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/6Fw4SQi.webp)

次に、 `systemctl restart chasquid`実行してサービスを再起動します。

## conf を git リポジトリにバックアップする

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Fier9uv.webp)

たとえば、次のように conf フォルダーを自分の github プロセスにバックアップします。

最初に専用倉庫を作成する

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ZSCT1t1.webp)

conf ディレクトリに入り、ウェアハウスに送信します

```
git init
git add .
git commit -m "init"
git branch -M main
git remote add origin git@github.com:wac-tax-key/conf.git
git push -u origin main
```

## 送信者を追加

走る

```
chasquid-util user-add i@wac.tax
```

送信者を追加できます

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/khHjLof.webp)

### パスワードが正しく設定されていることを確認する

```
chasquid-util authenticate i@wac.tax --password=xxxxxxx
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/e92JHXq.webp)

ユーザーを追加すると、 `chasquid/domains/wac.tax/users`更新されます。忘れずに倉庫に提出してください。

## DNS 追加 SPF レコード

SPF ( Sender Policy Framework ) は、電子メール詐欺を防止するために使用される電子メール検証技術です。

送信者の IP アドレスが、それが主張するドメイン名の DNS レコードと一致することを確認することで、メール送信者の身元を確認し、詐欺師が偽のメールを送信するのを防ぎます。

SPF レコードを追加すると、メールがスパムとして識別されるのを可能な限り防ぐことができます。

ドメイン ネーム サーバーが SPF タイプをサポートしていない場合は、TXT タイプのレコードを追加するだけです。

たとえば、 `wac.tax`の SPF は次のようになります。

`v=spf1 a mx include:_spf.wac.tax include:_spf.google.com ~all`

`_spf.wac.tax`の SPF

`v=spf1 a:smtp.wac.tax ~all`

ここに`include:_spf.google.com`があることに注意してください。これは、後で`i@wac.tax`を Google メールボックスの送信アドレスとして設定するためです。

## DNS 構成 DMARC

DMARCは（Domain-based Message Authentication, Reporting & Conformance）の略です。

これは、SPF バウンスをキャプチャするために使用されます (設定エラーが原因であるか、誰かがあなたになりすましてスパムを送信している可能性があります)。

TXT レコード`_dmarc`を追加し、

内容は以下の通り

```
v=DMARC1; p=quarantine; fo=1; ruf=mailto:ruf@wac.tax; rua=mailto:rua@wac.tax
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/k44P7O3.webp)

各パラメータの意味は次のとおりです。

### p (ポリシー)

SPF (Sender Policy Framework) または DKIM (DomainKeys Identified Mail) 検証に失敗した電子メールの処理方法を示します。 p パラメータは、次の 3 つの値のいずれかに設定できます。

* none: アクションは実行されず、確認結果のみが電子メール レポート メカニズムを通じて送信者にフィードバックされます。
* 検疫: 検証に合格しなかったメールをスパム フォルダに入れますが、メールを直接拒否することはありません。
* reject: 検証に失敗したメールを直接拒否します。

### fo (失敗オプション)

レポート メカニズムによって返される情報の量を指定します。次のいずれかの値に設定できます。

* 0: すべてのメッセージの検証結果を報告する
* 1: 検証に失敗したメッセージのみを報告する
* d: ドメイン名検証の失敗のみを報告する
* s: SPF 検証の失敗のみを報告する
* l: DKIM 検証の失敗のみを報告する

### ルア＆ラフ

* rua (集計レポートのレポート URI): 集計レポートを受信するための電子メール アドレス
* ruf (フォレンジック レポートのレポート URI): 詳細なレポートを受け取るための電子メール アドレス

## MX レコードを追加してメールを Google Mail に転送する

ユニバーサル アドレスをサポートする無料の企業用メールボックス (キャッチオール、このドメイン名に送信されたすべてのメールを受信でき、プレフィックスの制限なし) が見つからなかったため、chasquid を使用してすべてのメールを Gmail メールボックスに転送しました。

**独自の有料ビジネス メールボックスをお持ちの場合は、MX を変更せず、この手順をスキップしてください。**

`conf/chasquid/domains/wac.tax/aliases`を編集し、転送メールボックスを設定します

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/OBDl2gw.webp)

`*`はすべての電子メールを示し、 `i`は上記で作成した送信ユーザーの電子メール アドレスのプレフィックスです。メールを転送するには、各ユーザーが行を追加する必要があります。

次に、MX レコードを追加します (下の図の最初の行に示されているように、ここでは逆引きドメイン名のアドレスを直接指定しています)。

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/7__KrU8.webp)

設定が完了したら、他のメール アドレスを使用して`i@wac.tax`と`any123@wac.tax`にメールを送信し、Gmail でメールを受信できるかどうかを確認できます。

そうでない場合は、chasquid のログを確認します ( `grep chasquid /var/log/syslog` )。

## Google Mail で i@wac.tax にメールを送信

Google Mail にメールが届いたので、当然、i.wac.tax@gmail.com ではなく`i@wac.tax`で返信したいと思っていました。

[https://mail.google.com/mail/u/1/#settings/accounts](https://mail.google.com/mail/u/1/#settings/accounts)にアクセスし、[別のメール アドレスを追加] をクリックします。

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/PAvyE3C.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/_OgLsPT.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/XIUf6Dc.webp)

次に、転送された電子メールで受信した確認コードを入力します。

最後に、デフォルトの送信者アドレスとして設定できます (同じアドレスで返信するオプションと一緒に)。

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/a95dO60.webp)

以上でSMTPメールサーバーの構築は完了ですが、同時にGoogle Mailを利用してメールの送受信を行います。

## テストメールを送信して、構成が成功したかどうかを確認します

`ops/chasquid`に入る

`direnv allow`を実行して依存関係をインストールします (以前のワンキー初期化プロセスで direnv がインストールされ、シェルにフックが追加されました)。

次に実行します

```
user=i@wac.tax pass=xxxx to=iuser.link@gmail.com ./sendmail.coffee
```

パラメータの意味は次のとおりです。

* ユーザー: SMTP ユーザー名
* pass: SMTP パスワード
* 宛先: 受信者

テストメールを送信できます。

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ae1iWyM.webp)

Gmail を使用してテスト メールを受信し、設定が成功したかどうかを確認することをお勧めします。

### TLS 標準暗号化

下の図に示すように、この小さなロックがあります。これは、SSL 証明書が正常に有効になったことを意味します。

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/SrdbAwh.webp)

次に、「元のメールを表示」をクリックします

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/qQQsdxg.webp)

### DKIM

下の図のように、Gmail の元のメール ページに DKIM が表示されます。これは、DKIM の構成が成功したことを意味します。

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/an6aXK6.webp)

元のメールのヘッダーにある Received を確認すると、送信者アドレスが IPV6 であることがわかります。これは、IPV6 も正常に構成されていることを意味します。
