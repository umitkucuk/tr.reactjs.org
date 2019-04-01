---
id: strict-mode
title: Strict Modu
permalink: docs/strict-mode.html
---

`StrictMode`, bir uygulamadaki olası sorunları vurgulamak için bir araçtır. `Fragment` gibi, `StrictMode` da görünür bir UI oluşturmaz. Alt elemanları için ek kontroller ve uyarılar başlatır.

> Not:
>
> Strict mod denetimleri yalnızca geliştirme modunda çalıştırılır; _üretim derlemesini etkilemezler_.

Uygulamanızın herhangi bir bölümü için strict modu etkinleştirebilirsiniz. Örneğin:
`embed:strict-mode/enabling-strict-mode.js`

Yukarıdaki örnekte, strict mod denetimleri `Header` ve `Footer` bileşenlerine karşı *çalıştırılmaz*. Bununla birlikte, `ComponentOne` ve `ComponentTwo`, tüm alt elemanları gibi, kontrollere sahip olacak.

`StrictMode`'un şu anda yardımcı olduğu konular:
* [Güvenilmez yaşam döngüleri olan bileşenleri belirleme](#identifying-unsafe-lifecycles)
* [Eski string ref API kullanımı hakkında uyarma](#warning-about-legacy-string-ref-api-usage)
* [Kullanımdan kaldırılmış findDOMNode kullanımı hakkında uyarma](#warning-about-deprecated-finddomnode-usage)
* [Beklenmeyen yan etkileri tespit etme](#detecting-unexpected-side-effects)
* [Eski context API'ını tespit etme](#detecting-legacy-context-api)

Ek fonksiyonellikler, gelecekteki React sürümleri ile eklenecek.

### Güvenilmez yaşam döngülerini belirleme {#identifying-unsafe-lifecycles}

[Bu blog gönderisinde](/blog/2018/03/27/update-on-async-rendering.html) açıklandığı gibi, bazı eski yaşam döngüsü metotları asenkron React uygulamalarında kullanım için güvenli değildir. Bununla birlikte, eğer uygulamanız üçüncü taraf kütüphaneleri kullanıyorsa, bu yaşam döngülerinin kullanılmadığından emin olmak zor olabilir. Neyse ki, strict mod bu konuda yardımcı olabilir!

Strict mod etkinleştirildiğinde, React güvensiz yaşam döngülerini kullanarak tüm sınıf bileşenlerinin bir listesini oluşturur ve bu bileşenler hakkındaki bilgileri içeren şöyle bir uyarı mesajı loglar:

![](../images/blog/strict-mode-unsafe-lifecycles-warning.png)

Strict mod tarafından belirlenen sorunların ele alınması _şimdi_, gelecekteki React sürümlerinde asenkron render etme işleminden yararlanmanızı kolaylaştıracak.

### Eski string ref API kullanımı hakkında uyarma {#warning-about-legacy-string-ref-api-usage}

Previously, React provided two ways for managing refs: the legacy string ref API and the callback API. Although the string ref API was the more convenient of the two, it had [several downsides](https://github.com/facebook/react/issues/1373) and so our official recommendation was to [use the callback form instead](/docs/refs-and-the-dom.html#legacy-api-string-refs).

React 16.3 added a third option that offers the convenience of a string ref without any of the downsides:
`embed:16-3-release-blog-post/create-ref-example.js`

Since object refs were largely added as a replacement for string refs, strict mode now warns about usage of string refs.

> **Note:**
>
> Callback refs will continue to be supported in addition to the new `createRef` API.
>
> You don't need to replace callback refs in your components. They are slightly more flexible, so they will remain as an advanced feature.

[Learn more about the new `createRef` API here.](/docs/refs-and-the-dom.html)

### Warning about deprecated findDOMNode usage {#warning-about-deprecated-finddomnode-usage}

React used to support `findDOMNode` to search the tree for a DOM node given a class instance. Normally you don't need this because you can [attach a ref directly to a DOM node](/docs/refs-and-the-dom.html#creating-refs).

`findDOMNode` can also be used on class components but this was breaking abstraction levels by allowing a parent to demand that certain children was rendered. It creates a refactoring hazard where you can't change the implementation details of a component because a parent might be reaching into its DOM node. `findDOMNode` only returns the first child, but with the use of Fragments, it is possible for a component to render multiple DOM nodes. `findDOMNode` is a one time read API. It only gave you an answer when you asked for it. If a child component renders a different node, there is no way to handle this change. Therefore `findDOMNode` only worked if components always return a single DOM node that never changes.

You can instead make this explicit by passing a ref to your custom component and pass that along to the DOM using [ref forwarding](/docs/forwarding-refs.html#forwarding-refs-to-dom-components).

You can also add a wrapper DOM node in your component and attach a ref directly to it.

```javascript{4,7}
class MyComponent extends React.Component {
  constructor(props) {
    super(props);
    this.wrapper = React.createRef();
  }
  render() {
    return <div ref={this.wrapper}>{this.props.children}</div>;
  }
}
```

> Note:
>
> In CSS, the [`display: contents`](https://developer.mozilla.org/en-US/docs/Web/CSS/display#display_contents) attribute can be used if you don't want the node to be part of the layout.

### Detecting unexpected side effects {#detecting-unexpected-side-effects}

Conceptually, React does work in two phases:
* The **render** phase determines what changes need to be made to e.g. the DOM. During this phase, React calls `render` and then compares the result to the previous render.
* The **commit** phase is when React applies any changes. (In the case of React DOM, this is when React inserts, updates, and removes DOM nodes.) React also calls lifecycles like `componentDidMount` and `componentDidUpdate` during this phase.

The commit phase is usually very fast, but rendering can be slow. For this reason, the upcoming async mode (which is not enabled by default yet) breaks the rendering work into pieces, pausing and resuming the work to avoid blocking the browser. This means that React may invoke render phase lifecycles more than once before committing, or it may invoke them without committing at all (because of an error or a higher priority interruption).

Render phase lifecycles include the following class component methods:
* `constructor`
* `componentWillMount`
* `componentWillReceiveProps`
* `componentWillUpdate`
* `getDerivedStateFromProps`
* `shouldComponentUpdate`
* `render`
* `setState` updater functions (the first argument)

Because the above methods might be called more than once, it's important that they do not contain side-effects. Ignoring this rule can lead to a variety of problems, including memory leaks and invalid application state. Unfortunately, it can be difficult to detect these problems as they can often be [non-deterministic](https://en.wikipedia.org/wiki/Deterministic_algorithm).

Strict mode can't automatically detect side effects for you, but it can help you spot them by making them a little more deterministic. This is done by intentionally double-invoking the following methods:

* Class component `constructor` method
* The `render` method
* `setState` updater functions (the first argument)
* The static `getDerivedStateFromProps` lifecycle

> Note:
>
> This only applies to development mode. _Lifecycles will not be double-invoked in production mode._

For example, consider the following code:
`embed:strict-mode/side-effects-in-constructor.js`

At first glance, this code might not seem problematic. But if `SharedApplicationState.recordEvent` is not [idempotent](https://en.wikipedia.org/wiki/Idempotence#Computer_science_meaning), then instantiating this component multiple times could lead to invalid application state. This sort of subtle bug might not manifest during development, or it might do so inconsistently and so be overlooked.

By intentionally double-invoking methods like the component constructor, strict mode makes patterns like this easier to spot.

### Detecting legacy context API {#detecting-legacy-context-api}

The legacy context API is error-prone, and will be removed in a future major version. It still works for all 16.x releases but will show this warning message in strict mode:

![](../images/blog/warn-legacy-context-in-strict-mode.png)

Read the [new context API documentation](/docs/context.html) to help migrate to the new version.
