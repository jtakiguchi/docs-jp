---
title: ジェスチャーイベント
---

<!-- toc -->

Polymerは、特別なユーザーインタラクションに向けて独自の「ジェスチャーイベント」をオプションでサポートしています。ジェスチャーイベントは、タッチ及びマウスの両環境で一貫したイベントを発生させるので、仕様の`mouse-`イベントや`touch-`イベントに代えて利用されることが推奨されます。これにより、タッチとマウスの両デバイス間で相互運用性が向上します。

**In general, use the standard `click` event instead of `tap` in mobile browsers.** The `tap`
event is included in the gesture event mixin for backwards compatibility, but it's no longer
required in modern mobile browsers.
{.alert .alert-info}

## ジェスチャーイベントの使用

ハイブリッド要素を使用する場合は、デフォルトでジェスチャーイベントがサポートされています。`Polymer.Element`をベースにクラススタイルで作成した要素では、`Polymer.GestureEventListeners`ミックスインをインポートすることで、ジェスチャサポートを明示的に追加する必要があります。

```html
<link rel="import" href="polymer/lib/mixins/gesture-event-listeners.html">

<script>
    class TestEvent extends Polymer.GestureEventListeners(Polymer.Element) {
      ...
</script>
```


ジェスチャーイベントにはいくつか追加の設定が必要なため、単純に一般的な`addEventListener`メソッドを用いてリスナーを追加することはできません。ジェスチャーイベントを監視(liseten)するには、：

*   ジェスチャーイベントの一つに[アノテーション付イベントリスナー](events#annotated-listeners)を使用する。
       
    ```html
    <div id="dragme" on-track="handleTrack">Drag me!</div>
    ```
    
    アノテーション付イベントリスナーを使用した場合、Polymerは自動的にジェスチャーイベントを特別に記録するようになります。
    

*   `Polymer.Gestures.addListener`/`Polymer.Gestures.removeListener`メソッドを使用する
    
    ```js
    Polymer.Gestures.addListener(this, 'track', e => this.trackHandler(e));
    ```
    
    この`Polymer.Gestures.addListener`メソッドを使用して、ホスト要素にリスナーを追加できます。

### ジェスチャーイベントのタイプ

サポートされるジェスチャーイベントのタイプは次のとおりです。各タイプについて簡単な説明と`event.detail`から取得できる詳細なプロパティを列挙して紹介します。：

* **down**：指/ボタンが下げられた
  * `x`：イベントのclientX座標
  * `y`：イベントのclientY座標
  * `sourceEvent`：`down`アクションを最初に発生させたDOMイベント
* **up**：指/ボタンが上げられた
  * `x`：イベントのclientX座標
  * `y`：イベントのclientY座標
  * `sourceEvent`：`up`アクションを最初に発生させたDOMイベント
* **tap**：ダウン＆アップが発生した
  * `x`：イベントのclientX座標
  * `y`：イベントのclientY座標
  * `sourceEvent`：`tap`アクションを最初に発生させたDOMイベント
* **track**：指/ボタンが下げながら動いた
  * `state`：トラッキング状態を示す文字列：
    * `start`：トラッキングが最初に検出された時に発生(指/ボタンが押され、事前に設定された距離の閾値を超えて移動した時）
    * `track`：トラッキングの最中に発生
    * `end`：トラッキングが終了した時に発生
  * `x`：イベントのclientX座標
  * `y`：イベントのclientY座標
  * `dx`：トラックイベントの開始時から水平方向にピクセル単位で生じた変化
  * `dy`：トラックイベントの開始時から垂直方向にピクセル単位で生じた変化
  * `ddx`：トラックイベントの終了時からから水平方向にピクセル単位で生じた変化
  * `ddy`：トラックイベントの終了時からから垂直方向にピクセル単位で生じた変化
  * `hover()`：現在ホバーされている要素を判断するために呼び出される可能性のあるメソッド

### 例

宣言的イベントリスナーの例 { .caption }

```html
<link rel="import" href="polymer/polymer-element.html">
<link rel="import" href="polymer/lib/mixins/gesture-event-listeners.html">

<dom-module id="drag-me">
  <template>
    <style>
      #dragme {
        width: 500px;
        height: 500px;
        background: gray;
      }
    </style>

    <div id="dragme" on-track="handleTrack">[[message]]</div>
  </template>

  <script>
    class DragMe extends Polymer.GestureEventListeners(Polymer.Element) {

      static get is() {return 'drag-me'}

      handleTrack(e) {
        switch(e.detail.state) {
          case 'start':
            this.message = 'Tracking started!';
            break;
          case 'track':
            this.message = 'Tracking in progress... ' +
              e.detail.x + ', ' + e.detail.y;
            break;
          case 'end':
            this.message = 'Tracking ended!';
            break;
        }
      }

    }
    customElements.define(DragMe.is, DragMe);
  </script>
</dom-module>
```

命令的なイベントリスナーの例 { .caption }

以下の例では、ホスト要素に対してリスナーを追加するのに`Polymer.Gestures.addListener`を使用しています。このようなケースでは、アノテーション付イベントリスナーで設定することができません。もし、リスナーがホスト要素またはシャドーDOMの子に対して設定されている場合には、通常、一度追加したイベントリスナーに関してその削除について心配する必要はありません。

動的に追加された子にイベントリスナーを設定する場合には、子を削除する時点で、`Polymer.Gestures.addListener`によって設定されたイベントリスナーを削除する必要があるかもしれません。これは、子要素がガベージコレクトされるようにするためです。

```html
<link rel="import" href="../../bower_components/polymer/polymer-element.html">
<link rel="import" href="../../bower_components/polymer/lib/mixins/gesture-event-listeners.html">

<dom-module id="drag-me-app">
  <template>
    <style>
      :host {
        border: 1px solid blue;
        background: gray;
      }
    </style>
    [[message]]
  </template>

  <script>
    class DragMeApp extends Polymer.GestureEventListeners(Polymer.Element) {
      static get is() { return 'drag-me-app'; }
      static get properties() {
        return {
          message: {
            type: String,
            value: "Select my text. I will track you."
          }
        };
      }
      constructor() {
        super();
        Polymer.Gestures.addListener(this, 'track', e => this.handleTrack(e));
      }
      handleTrack(e) {
        switch(e.detail.state) {
          case 'start':
            this.message = 'Tracking started!';
            break;
          case 'track':
            this.message = 'Tracking in progress... ' +
              e.detail.x + ', ' + e.detail.y;
            break;
          case 'end':
            this.message = 'Tracking ended!';
            break;
        }
      }
    }
    customElements.define(DragMeApp.is, DragMeApp);
  </script>
</dom-module>

```

## ジェスチャーとスクロールの方向

特定のジェスチャーを監視(listen)した場合には、タッチ入力に関してスクロール方向が制御されます。例えば、`track`イベントのリスナーを持つノードは、デフォルトでスクロールが禁止されます。要素は、`this.setScrollDirection(direction, node)`でスクロール方向を上書きすることができます。
引数`direction`には`x`、`y`、`none`、`all`のいずれかになり、引数`node`はデフォルトで`this`になります。