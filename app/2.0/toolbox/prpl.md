---
title: PRPL Pattern
---

<!-- toc -->

配信を最適化するために、Toolboxは_PRPLパターン_を使用します。：

*  重要なリソースを初期ルート(initial route)へプッシュ(**P**ush)します。
*  初期ルートをレンダリング(**R**ender)します。
*  残りのルートをプレキャッシュ(**P**re-cache)する。
*  必要に応じて残りのルートをレイジーロード(**L**azy-load)し生成します。

これらを実行するために、サーバーはアプリケーションの各ルートで必要とされるリソースを識別できるようにする必要があります。リソースを一つのダウンロード単位にバンドルする代わりに、HTTP2プッシュを使用して、要求されたルートのレンダリングに必要なリソースを個別に配信します。

サーバーとサービスワーカーは、非アクティブなルートに対するリソースを事前にキャッシュするために連動して動作します。

ユーザーがルートを切り替えると、アプリケーションは未キャッシュだが必要されたリソースをレイジーロード(lazy-load)し、要求されたビューが生成されます。

## アプリケーションの構造

現在、Polymer CLIおよびリファレンスサーバーは、以下のような構造のシングルページアプリケーション（SPA）をサポートしています。(訳注：下の図を参照しながら説明を読んで下さい。)：

-   すべての有効なルートから供給されるアプリケーションのメインの_エントリーポイント_(index.html)。このファイルは可能な限り小さくすべきです。異なるURLから提供されるため、何度もキャッシュされるかもしれないからです。エントリポイント内のすべてのリソースのURLは、トップレベルではないURLからも提供される可能性があるので、絶対パスを使用する必要があります。
-   _シェル_やapp-shellには、トップレベルのアプリケーションロジックやルーターなどが含まれます。
-   レイジーロードされるアプリケーションの_フラグメント_。フラグメントとして、個々のビューのコードやレイジーロード可能なその他コード（例えば、ユーザーがアプリケーションとやりとりするまで表示されないメニューのように、最初の描画に不要なメインアプリケーションの一部）を表すことができます。シェルには、必要に応じてフラグメントを動的にインポートする責任があります。

下の図は、単純なアプリケーションのコンポーネントを示しています。：

![diagram of an app that has two views, which have both individual and shared dependencies](/images/2.0/toolbox/app-build-components.png)


この図では、実線は静的な依存関係を表し、`<link>`タグと`<script>`タグを使ってファイル内で識別される外部リソースを表しています。点線は、動的な依存関係や読み込み要求のあった依存関係(シェルからの要求に応じて読み込まれるファイル)を表現しています。

ビルドプロセスでは、これらのすべての依存関係に関するグラフを構築し、サーバーはこの情報を利用してファイルを効率的に提供します。また、HTTP2のプッシュをサポートしていないブラウザ向けに、vulcanizeされた各種バンドルも生成します。

### アプリケーションのエントリーポイント

エントリーポイントは、シェルをインポートしてインスタンス化するだけでなく、必要な場合にはポリフィルをロードします。

エントリーポイントの主な考慮事項は次のとおりです。；

-   静的な依存は最小限に抑えます。これはapp-shellそのものの依存より遥かに少ないものです。
-   必要であれば条件に応じてポリフィルをロードします。
-   すべての依存関係に対して絶対パスを使用します。

Polymer CLIを使用してApp Toolboxプロジェクトを生成すると、新しいプロジェクトにはエントリーポイント`index.html`が含まれます。ほとんどのプロジェクトでは、このファイルを書き換える必要はないはずです。

### アプリケーションのシェル(App Shell)

シェルはルーティングを担当し、通常はアプリケーションのメインナビゲーションUIが含まれます。

アプリケーションは、フラグメントが必要とされるまで読み込まれるのを遅延するために、`importHref`を呼び出す必要があります。例えば、ユーザーが新しいルートに変更すると、そのルートに関連付けられたフラグメントがインポートされます。これにより、サーバーへの新たなリクエストが送られるか、あるいは、キャッシュからリソースがロードされるだけかもしれません。

importHrefの例(クラススタイル要素) {.caption}

```js
// get a URL relative to this element
let resolvedUrl = this.resolveUrl('list-view.html');

// import the file
Polymer.importHref(
    resolvedUrl,
    null,  /* callback for successful load -- usually not needed */
    this._importFailedCallback.bind(this), /* for example, display 404 page */
    true); /* make import async */
```

importHrefの例(ハイブリッド要素) {.caption}

```js
var resolvedPageUrl = this.resolveUrl('my-' + page + '.html');
this.importHref(resolvedPageUrl,
    null,
    this._importFailedCallback,
    true);
```

シェル（静的な依存関係を含む）には、最初の描画に必要なものが全て含まれている必要があります。

## ビルドの出力

By default, the Polymer CLI build process produces an unbundled build designed for server/browser combinations that support HTTP/2 and HTTP/2 server push to deliver the resources the browser needs for a fast first paint while optimizing caching.

To generate a bundled build, designed to minimize the number of round-trips required to get the application running on server/browser combinations that don't support server push, you will need to pass a command line option or configure your [polymer.json file](polymer-json) appropriately.

複数のビルドを生成した場合、サーバーロジックでブラウザごとに適切なビルドを供給する必要があります。

### Bundled build

HTTP2のプッシュを扱わないブラウザの場合、`--bundle`フラグバンドルのセットを出力します。：
一方はシェル用のバンドルで、もう一方は各フラグメント用のバンドルです。
下の図は、単純なアプリケーションがどのようにバンドルされるかを示しています。：

![diagram of the same app as before, where there are 3 bundled dependencies](/images/2.0/toolbox/app-build-bundles.png)

二つ以上のフラグメントによって共有された依存関係はすべて、シェルとその静的な依存関係にバンドルされています。

各フラグメントと_共有されていない_静的な依存関係は、一つにバンドルされています。サーバーは、ブラウザに応じて、適切なバージョンのフラグメント（bundledまたはunbundled）を返す必要があります。つまり、_シェルのコードは、bundledかunbundledか知らなくても_、`detail-view.html`をレイジーロードできるということです。依存関係が最も効率な方法でロードされるかは、サーバーとブラウザ次第です。


## HTTP/2とHTTP/2サーバープッシュの背景

HTTP/2を使用すると、単一の接続上で多重ダウンロードを行うことができるため、複数の小さなファイルをより効率的にダウンロードできます。

HTTP/2のサーバープッシュにより、サーバーはブラウザからの要求を先取りしてリソースを送信できます。

HTTP/2のサーバープッシュのダウンロード速度をどのように上げるかについては、ブラウザがスタイルシートのリンクを使ってどのようにHTMLファイルを読み出しているか考察してみてくだい。

HTTP/1では：

*   ブラウザがHTMLファイルを要求します。
*   サーバがHTMLファイルを返し、ブラウザが解析を開始します。
*   ブラウザは`<link rel="stylesheet">`タグを見つけると、スタイルシートの新しい要求を開始します。
*   ブラウザはスタイルシートを受信します。


HTTP/2のプッシュを利用すると：

*   ブラウザがHTMLファイルを要求します。
*   サーバーはHTMLファイルを返すと同時にスタイルシートをプッシュします。 
*   ブラウザがHTMLの解析を開始します。`<link rel="stylesheet">`が見つかりますが、スタイルシートはすでにキャッシュに入っています。

この最も単純なケースでは、HTTP/2のサーバープッシュはHTTPのリクエスト/レスポンスを一つなくすことができました。

HTTP/1では、開発者はリソースをまとめてバンドルすることで、ページのレンダリングに必要なHTTPリクエストの数を減らします。ただし、バンドルすることでブラウザのキャッシュ効率が低下する可能性があります。仮に各ページのリソースが一つのバンドルに結合されているとしたら、個々のページはそれぞれ独自のバンドルを取得することになり、ブラウザは共有リソースを識別することができません。

HTTP/2とHTTP/2のサーバープッシュの組み合わせは、リソースをバンドルすることなく、バンドル処理の利点（遅延時間の短縮）を享受できます。リソースを分割しておくことで、これらは効率的にキャッシュされページ間で共有することができます。
