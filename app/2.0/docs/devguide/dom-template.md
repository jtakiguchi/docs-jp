---
title: DOMテンプレート
---

<!-- toc -->

多くの要素は、その機能の実装にDOM要素のサブツリーを利用しています。DOMテンプレートを利用することで、要素にDOMのサブツリーを簡単に作成することができます。

デフォルトでは、要素にDOMテンプレートを追加すると、Polymerは要素にShadow Rootを作成し、テンプレートをShadow Tree内部に複製(clone)します。

また、DOMテンプレートによってデータバインディングと宣言型イベントハンドラが利用できるようになります。

## DOMテンプレートの概要

Polymer provides three basic ways to specify a DOM template:

-   `<dom-module>` element. This allows you to specify the template entirely in markup,
    which is most efficient for an element defined in an HTML import.
-   Provide a string template. This works well for elements defined entirely in JavaScript
    (for example, in an ES6 module).
-   Retrieve or generate your own template element. This allows you to define how the template is
    retrieved or constructed, and can be used to modify a superclass template.

Polymer provides a default `template` getter that retrieves a template from the element's
`<dom-module>`. You can override this getter to provide a string template or a generated template
element.

### Specify a template using dom-module

To specify an element's DOM template using a `<dom-module>`:
`<dom-module>`を使って要素のDOMテンプレートを記述するには：

1.  要素の名前と同名の`id`属性を持つ`<dom-module>`要素を作成します。
2.  `<dom-module>`の内部に`<template>`要素を作成します。
3.  Give the element a static `is` getter that matches the element's name. Polymer uses this to retrieve the `<dom-module>` for the element.

Polymerは、このテンプレートの内容を要素のShadow DOM内部に複製(clone)します。

例: { .caption }

```html
<dom-module id="x-foo">

  <template>I am x-foo!</template>

  <script>
    class XFoo extends Polymer.Element {
      static get is() { return  'x-foo' }
    }
    customElements.define(XFoo.is, XFoo);
  </script>

</dom-module>
```

この例では、DOMテンプレートとその要素を定義するJavaScriptが同じファイルにあります。これらを別々のファイルに分割して配置することもできますが、DOMテンプレートの解析は要素が定義される前に行われる必要があります。

**注意**：テスト環境を除き、要素はメインドキュメントの外で定義すべきです。メインドキュメント内における要素の定義に関する注意事項は、[main document definitions](registering-elements#main-document-definitions)を参照してください。
{.alert .alert-info}

### Specify a string template

As an alternative to specifying the element's template in markup, you can specify a string template
by creating a static `template` getter that returns a string.

This getter is called _once_, when the first instance of the element is upgraded.

```js
class MyElement extends Polymer.Element {

  static get template() {
    return `<style>:host { color: blue; }</style>
       <h2>String template</h2>
       <div>I've got a string template!</div>`
  }
}
customElements.define('my-element', MyElement);
```

When using a string template, the element doesn't need to provide an `is` getter (however, the tag
name still needs to  be  passed as the first argument to `customElements.define`).

### Inherited templates {#inherited-templates}

An element that extends another Polymer element can inherit its template. If the element doesn't
provide its own DOM template (using either a `<dom-module>` or a string template), Polymer uses the
same template as the superclass, if any.

You can also modify a superclass template by defining a `template` getter that returns a modified
template element. If you're going to modify the superclass template, there are a couple of
important rules:

-   Don't modify the superclass template in place; make a copy before modifying.
-   If you're doing anything expensive, like copying or modifying an existing template,
    you should memoize the modified template so you don't have to regenerate it when the
    getter is called.

The following example shows a simple modification based on a parent template:

```js
(function() {
  let memoizedTemplate;

  class MyExtension extends MySuperClass {
    static get template() {
      if (!memoizedTemplate) {
        // create a clone of superclass template (`true` = "deep" clone)
        memoizedTemplate = MySuperClass.template.cloneNode(true);
        // add a node to the template.
        let div = document.createElement('div');
        div.textContent = 'Hello from an extended template.'
        memoizedTemplate.content.appendChild(div);
      }
      return memoizedTemplate;
    }
  }

})();
```

The following example shows how an element could wrap a superclass template with its own
template.

```html
<dom-module id="my-ext">
    <template>
      <h2>Extended template</h2>
      <!-- superclass template will go here -->
      <div id="footer">
        Extended template footer.
      </div>
    </template>
  <script>
    (function() {
      let memoizedTemplate;

      class MyExtension extends MySuperClass {

        static get is() { return 'my-ext'}
        static get template() {
          if (!memoizedTemplate) {
            // Retrieve this element's dom-module template
            memoizedTemplate = Polymer.DomModule.import(this.is, 'template');

            // Clone the contents of the superclass template
            let superTemplateContents = document.importNode(MySuperClass.template.content, true);

            // Insert the superclass contents
            let footer = memoizedTemplate.content.querySelector('#footer');
            memoizedTemplate.content.insertBefore(superTemplateContents, footer);
          }
          return memoizedTemplate;
        }
      }
      customElements.define(MyExtension.is, MyExtension);
    })();
  </script>
</dom-module>
```

### Elements with no shadow DOM

To create an element with no shadow DOM, don't specify a DOM template (either using a `<dom-module>`
or by overriding the `template` getter), then no shadow root is created for the element.

If the element is extending another element that has a DOM template and you don't want a DOM template,
define a `template` getter that returns a falsy value.

### URLs in templates {#urls-in-templates}

A relative URL in a template may need to be relative to an application or to a specific component.
For example, if a component includes images alongside an HTML import that defines an element, the
image URL needs to be resolved relative to the import. However, an application-specific element may
need to include links to URLs relative to the main document.

By default, Polymer **does not modify URLs in templates**, so all relative URLs are treated as
relative to  the main document URL. This is because when the template content is cloned and added
to the main document, the browser evaluates the URLs  relative to the document (not to the original
location of the template).

To ensure URLs resolve properly, Polymer provides two properties that can be used in data bindings:

| Property | Description |
| -------- | ----------- |
| `importPath` | A static getter on the element class that defaults to the element HTML import document URL and is overridable. It may be useful to override `importPath` when an element's template is not retrieved from a `<dom-module>` or the element is not defined using an HTML import. |
| `rootPath` | An instance property set to the value of `Polymer.rootPath` which is globally settable and defaults to the main document URL. It may be useful to set `Polymer.rootPath` to provide a stable application mount path when using client side routing. |


Relative URLs in styles are automatically re-written to be relative to the `importPath` property.
Any URLs outside of a `<style>` element should be bound using `importPath` or
`rootPath` where appropriate. For example:

```html
<img src$="[[importPath]]checked.jpg">
```

```html
<a href$="[[rootPath]]users/profile">View profile</a>
```

The `importPath` and `rootPath` properties are also supported in Polymer 1.9+, so they can be used
by hybrid elements.



## 静的ノードマップ {#node-finding}

Polymerは、要素がDOMテンプレートを初期化する際に、ノードIDの静的マップを作成し、頻繁に使用されるノードに手軽にアクセスできるようにします。手動でクエリを記述する必要はありません。要素のテンプレートに`id`付きで記述されたノードはそれぞれ、`id`によってハッシュ化され`this.$`として格納されます。


The `this.$` hash is created when the shadow DOM is initialized. In the `ready` callback, you must
call `super.ready()` before accessing `this.$`.

**注意：**データバインディングを使用して動的に作成されたノード(`dom-repeat`テンプレートや`dom-if`テンプレートによるものが含まれます)は、ハッシュ`this.$`には追加されません。ハッシュには、静的に作成されたローカルDOMノード(つまり要素の最も外側のテンプレートに定義されたノード)だけが含まれます。
{.alert .alert-info}

例: { .caption }

```html
<dom-module id="x-custom">

  <template>
    Hello World from <span id="name"></span>!
  </template>

  <script>
    class MyElement extends Polymer.Element {
      static get is() { return  'x-custom' }
      ready() {
        super.ready();
        this.$.name.textContent = this.tagName;
      }
    }
  </script>

</dom-module>
```

要素のShadow DOM内に動的に生成されたノードを配置する場合、標準DOMの`querySelector`メソッドを利用してください。

<code>this.shadowRoot.querySelector(<var>selector</var>)</code>



## Remove empty text nodes {#strip-whitespace}


Add the `strip-whitespace` boolean attribute to a template to remove
any **empty** text nodes from the template's contents. This can result in a
minor performance improvement.

**What's an empty node?** `strip-whitespace` removes only text nodes that occur between
elements in the template and are _empty_ (that is, they only contain whitespace characters).
These nodes are created when two elements in the template are separated by whitespace (such as
spaces or line breaks). It doesn't remove any whitespace from inside elements.
{.alert .alert-info}

With empty text nodes:


```html
<dom-module id="has-whitespace">
  <template>
    <div>Some Text</div>
    <div>More Text</div>
  </template>
  <script>
    class HasWhitespace extends Polymer.Element {
      static get is() { return  'has-whitespace' }
      ready() {
        super.ready();
        console.log(this.shadowRoot.childNodes.length); // 5
      }
    }
    customElements.define(HasWhitespace.is, HasWhitespace);
  </script>
</dom-module>
```

There are five nodes in this element's shadow tree because of the whitespace surrounding the `<div>`
elements. The five child nodes are:

  text node
  `<div>Some Text</div>`
  text node
  `<div>More Text</div>`
  text node

Without empty text nodes:


```html
<dom-module id="no-whitespace">
  <template strip-whitespace>
    <div>Some Text</div>
    <div>More Text</div>
  </template>
  <script>
    class NoWhitespace extends Polymer.Element {
      static get is() { return  'no-whitespace' }
      ready() {
        super.ready();
        console.log(this.shadowRoot.childNodes.length); // 2
      }
    }
    customElements.define(NoWhitespace.is, NoWhitespace);
  </script>
</dom-module>
```

Here, the shadow tree contains only the two `<div>` nodes:

`<div>Some Text</div><div>More Text</div>`

Note that the whitespace _inside_ the `<div>` elements isn't affected.

## Preserve template contents

Polymer performs one-time processing on your DOM template. For example:

-   Parsing and removing binding annotations.
-   Parsing and removing markup for declarative event listeners.
-   Caching and removing the contents of nested templates for better performance. 

This processing removes the template's original contents (the `content` property will be undefined). If you want
to access the contents of a nested template, you can add the `preserve-content` attribute to the
template.

Preserving the contents of a nested template means it **won't have any Polymer features like
data bindings or declarative event listeners.** Only use this when you want to manipulate the
template yourself, and you don't want Polymer to touch it.

This is a fairly rare use case.

```html
<dom-module id="custom-template">
  <template>
    <template id="special-template" preserve-content>
      <div>I am very special.</div>
    </template>
  </template>
  <script>
    class CustomTemplate extends Polymer.Element {

      static get is() { return  'custom-template' }

      ready() {
        super.ready();
        // retrieve the nested template
        let template = this.shadowRoot.querySelector('#special-template');

        //
        for (let i=0; i<10; i++) {
          this.shadowRoot.appendChild(document.importNode(template.content, true));
        }
      }
    }

    customElements.define(CustomTemplate.is, CustomTemplate);
  </script>
</dom-module>
```

## Customize DOM initialization

There are several points where you can customize how Polymer initializes your element's DOM. You
can customize how the shadow root is created by creating it yourself. And you can override the
`_attachDom` method to change how the the DOM tree is added to your element: for example, to
stamp into light DOM instead of shadow DOM.

### Create your own shadow root

In some cases, you may want to create your own shadow root. You can do this by creating a shadow root
before calling `super.ready()`—or before the `ready` callback, such as in the constructor.

```js
constructor() {
  super();
  this.attachShadow({mode: 'open', delegatesFocus: true});
}
```

You can also override the `_attachDom` method:

```js
_attachDom(dom) {
  this.attachShadow({mode: 'open', delegatesFocus: true});
  super._attachDom(dom);
}
```

### Stamp templates in light DOM

You can customize how the DOM is stamped by overriding the `_attachDom` method. The method takes a
single argument, a `DocumentFragment` containing the DOM to be stamped. If you want to stamp the
template into light DOM, simply add an override like this:

```js
_attachDom(dom) {
  this.appendChild(dom);
}
```

When you stamp the DOM template to light DOM like this, data bindings and declarative event listeners
work as usual, but you cannot use shadow DOM features, like `<slot>` and style encapsulation.

A template stamped into light DOM shouldn't contain any `<style>` tags. Styles can be applied by an
enclosing host element, or at the document level if the element isn't used inside another element's
shadow DOM.
