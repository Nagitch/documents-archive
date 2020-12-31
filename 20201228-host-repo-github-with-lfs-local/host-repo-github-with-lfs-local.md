## 背景

Web等のプロジェクトではリポジトリをクラウドホスティングするのは一般的ですが、ゲーム系のプロジェクトは大容量になりがちなため、まともな会社ならともかく個人で開発しているとクラウドにホスティングする手段がなかったりします。
例えばGitHubは、無料枠で1 GBのストレージ, １ヶ月あたり1GBの帯域となっていて、個人レベルでも簡単に使い切ってしまうでしょう。$5/月で、１ヶ月50 GBの帯域と 50 GBのストレージに拡張できますが、はじめのプロジェクト全体のPushとPC入れ替えの時は手が震えそうですね。
参照: [Git Large File Storage の支払いについて](https://docs.github.com/ja/free-pro-team@latest/github/setting-up-and-managing-billing-and-payments-on-github/about-billing-for-git-large-file-storage)

そうなると大概はローカルのみで管理することになりますが、当然ながらPCが死ぬとプロジェクトも死ぬので心中穏やかではありません。（なので、21世紀なのにまめに手動バックアップせざるを得ない）

そしてなにより、GitHubに<b>草が生えません</b>。進捗はあるのに！

その悲しみを軽減するため、リモートリポジトリはGitHubだけど、LFSだけをローカル(Docker)にホスティングするという折衷案を実現したので手順を記載します。

ちなみに大容量になりがちなゲーム系プロジェクトを例にしてますが、どのようなプロジェクトでも適用できる想定です。

## 注意事項・免責

- 複数人で開発することは考慮していません。
- Git, Dockerがわかる方前提で書いています。
- 手順の説明はWindows, Docker for Windows + WSL2 バックエンドの前提です。ご了承ください。
- セキュリティについて考慮していないので、公的なプロジェクトで導入するのはおすすめしません。有事の責任はとれません。


## FAQ

> LFSをお安くクラウドにホスティングしたい

現実的な価格でAmazon S3 にホスティング（LFS APIサーバー自体はローカル、ストレージはS3）する方法がこちらで説明されています。
[Git LFSをAmazon S3でいい感じにする話](https://ydkk.hateblo.jp/entry/2017/12/07/120000)

> 草が生えなくてよいので、LFSをローカルホスティングしたい

GitLabをLFS込みでローカルホスティング(Docker)するのが手っ取り早いと思います。
またUE4はGitはデファクトじゃないっぽい（私見）ので、PerforceやSVNを使うと良さそう。
私はGitに慣れているので使っているのもあります。

> 無料でできる方法ですか？

はい。


## 実現方法の解説

※手順のみ知りたい方は飛ばしてOK

構成が少々複雑になるので説明します。
まずGitHub(with Git LFS)を利用した通常の管理の場合。

![hoting-github-standard-1.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/58251/20bb7dd1-9d36-abcb-3e3c-359afb4b49b3.png)
図：GitHubを利用した通常の管理

LFSで管理されるファイルは一意のoidが与えられ、リポジトリから参照されます。その仕組みはローカルでもリモート(GitHub)でも同じなのですが、
リポジトリサーバーの裏ではLFSサーバーが動いており、WebAPI経由でoidをもとにオブジェクトをやりとりしています。
参考： [git-lfsの仕様(サーバ側)を個人的に解説してみる](https://jyn.jp/git-lfs-api/)

![hoting-github-standard-2.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/58251/2e98553c-9c6a-aae1-19a3-d9ed1de68240.png)
図：解決案

その仕組みと、GitHubでは `.lfsconfig` でLFSのホスト先を指定できる機能を利用し、DockerでLFSサーバーをホストします。
LFSサーバーアプリケーションには、[lfs-test-server](https://github.com/git-lfs/lfs-test-server) を利用します。（Git LFS クライアントのテスト用に作られたサーバープログラム）

カンの良い方は気付いたかもしれませんが、この構成だとCドライブが壊れたらバックアップを失ってしまいます。
そのためDockerイメージをDドライブ等に保存する手順を後述します。


## 手順

プロジェクトはすでにローカルリポジトリが用意されている前提で書いています。
またDockerはWSL2バックエンドになっている前提です。

### 準備

Docker for Windowsでコンテナ（仮想環境）を立ち上げられる状態にしておいてください。（これが少しめんどくさいのですが、、詳細は割愛）

参考: [Windows 10 Home で WSL 2 + Docker を使う](https://qiita.com/KoKeCross/items/a6365af2594a102a817b)


### (オプション) Dockerイメージ・コンテナをDドライブ等に保存するように設定する

標準の設定のままだと、ローカルホスティングの環境はCドライブ下で構成されるるため、壊れたらバックアップを失ってしまいます。
こちらの記事の手順に従いDockerイメージ・コンテナをDドライブ等に保存するように設定します。
[WSL2 Dockerのイメージ・コンテナの格納先を変更したい (WSL2のvhdxファイルを移動させたい)](https://qiita.com/neko_the_shadow/items/ae87b2480345152bc3cb)

これによりCドライブが壊れても、GitHubとDドライブが生きていればリポジトリを復元できます。


### LFSサーバーアプリケーションをDockerで起動する

GitHubのLFSサーバーソース・バイナリは公開されていないのですが、
Git LFS クライアントのテスト用に作られたサーバープログラム [lfs-test-server](https://github.com/git-lfs/lfs-test-server) が公開されているので利用します。

やってしまいがちですが、CドラにCloneしてWSL2とVolumeを共有し走らせるとパフォーマンスが悪いため、WSL2の環境下のみでClone＆実行するように注意します。手順は以下。

Ubuntu（任意のWSL2 distro）を立ち上げる

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/58251/cfab4098-ed2a-e485-a4e3-021adb9b81e6.png)


homeディレクトリでlfs-test-serverをクローン
※え？ここってCドラじゃない？と思った方。ややこしいですが、ソースはここにCloneするけどDockerコンテナはDドライブに保存されるので大丈夫です。

```
cd ~
git clone https://github.com/git-lfs/lfs-test-server.git
cd lfs-test-server
```

Dockerfileが入ってるので利用します。（この場合は、開発者用の環境になります。ちゃんとしたい方はバイナリビルド＆実行してください。。マルチステージビルドで環境整えられればより良いかも）

lfs-test-serverは最低限、管理画面のadminユーザー設定をENVで与えなければいけないのと、
私はコマンド一発で立ち上げられるようにしたいので、プロジェクトルートに `docker-compose.yml` と `.env` ファイルを追加します。（設定は任意で変更してください）

```yaml:docker-compose.yml
version: "3"
services:
  lfs-test-server:
    container_name: "lfs-test-server"
    build: .
    ports:
      - "8080:8080"
      - "1080:1080"
    environment:
      - LFS_ADMINUSER=${LFS_ADMINUSER- }
      - LFS_ADMINPASS=${LFS_ADMINPASS- }
```

```shell:.env
# LFS_LISTEN            # The address:port the server listens on, default: "tcp://:8080"
# LFS_HOST              # The host used when the server generates URLs, default: "localhost:8080"
# LFS_METADB            # The database file the server uses to store meta information, default: "lfs.db"
# LFS_CONTENTPATH       # The path where LFS files are store, default: "lfs-content"
LFS_ADMINUSER="admin"   # An administrator username, default: not set
LFS_ADMINPASS="admin"   # An administrator password, default: not set
# LFS_CERT              # Certificate file for tls
# LFS_KEY               # tls key
# LFS_SCHEME            # set to 'https' to override default http
# LFS_USETUS            # set to 'true' to enable tusd (tus.io) resumable upload server; tusd must be on PATH, installed separately
# LFS_TUSHOST           # The host used to start the tusd upload server, default "localhost:1080"
```

imageをビルド＆コンテナを立ち上げます。

```
docker-compose up --build
```

しばらく待って、エラー等ないことが確認出来たら次に進みます。


### lfs-test-server の管理画面からユーザーを追加する

GitHubでいうところのユーザーを管理画面から追加する必要があるので、追加します。
上記の手順のままの場合、 http://localhost:8080/mgmt にアクセス。
.envで設定した管理ユーザー名・パスワードでログインします。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/58251/b13a68d3-f798-e1e4-2ba1-dba6089a8f50.png)

Userメニューからユーザーを追加します。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/58251/fea9654e-450a-227e-d33f-428f37f8e1a1.png)

これでLFSサーバーは準備完了です。


### プロジェクトにLFSの設定ファイルを追加する

すでにローカルリポジトリのLFSの設定が終わっていれば、次の `.lfsconfig` を追加してリモートのLFSをローカルのlfs-test-serverに向けます。

```.lfsconfig
[lfs]
	url = http://localhost:8080
```

`.gitattributes`, `.gitignore` をまだ追加していなければ追加します。
[Unityはこちら](https://github.com/Nagitch/unity-lfs-host-local-example)、[UE4はこちら](https://github.com/Nagitch/ue4-lfs-host-local-example) にリポジトリのサンプルがあるので良かったら利用してください。

ここで一旦、 `.lfsconfig`, `.gitattributes`, `.gitignore` だけ先にpushしておきます。（ホスティング先をlocalhostに向けられないかもしれないので念のため）

```shell
git add .lfsconfig .gitattributes .gitignore
git commit -m "add configs"
git push origin master
```

以降、PushしたらLFSオブジェクトはlfs-test-serverで管理されます。

### lfs-test-serverにホストされていることを確認する

LFSにPushされるとlfs-test-serverのログにoidが残るので、GitHubのリポジトリのLFSオブジェクトと突き合せれば成功したらしいことがわかります。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/58251/9be3b594-b2be-efdb-8091-38a45e6612e2.png)


![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/58251/3e8d8068-02ad-3b44-dfd9-88cce36be468.png)


![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/58251/2ac0d9ac-e1b3-63bd-5fd5-1fcc0d7ab93a.png)


## 所感など

構成の検討含め、かなり面倒な作業だったけど、草を生やすことは日々のモチベーションにつながるのでやる価値はあったと思っています。

あとは、当然ながら大きいファイルはコミットするたびにLFSとローカルの `.git` 以下に保存されていってストレージを食いつぶしていく問題があるので、
リリースのタイミングで手動コピーして一度環境を潰す。とかしないといけなさそう。
（どの構成管理ツールでも同じ問題はある気がするけど、どうしているのか。。）
