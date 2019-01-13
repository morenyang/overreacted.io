---
title: setState 是如何知道该做什么的？
date: '2018-12-09'
spoiler: 依赖注入是个很好的解决方法，只要你不去研究它。
---

当你使用在组件中调用 `setState` 时，你认为会发生什么？

```jsx{11}
import React from 'react';
import ReactDOM from 'react-dom';

class Button extends React.Component {
  constructor(props) {
    super(props);
    this.state = { clicked: false };
    this.handleClick = this.handleClick.bind(this);
  }
  handleClick() {
    this.setState({ clicked: true });
  }
  render() {
    if (this.state.clicked) {
      return <h1>Thanks</h1>;
    }
    return (
      <button onClick={this.handleClick}>
        Click me!
      </button>
    );
  }
}

ReactDOM.render(<Button />, document.getElementById('container'));
```

当然，React 会使用新的状态 `{ clicked: true }` 来重新渲染组件，并返回与之匹配的元素 `<h1>Thanks</h1>` 来更新DOM。

这看起来直截了当。但是，稍等一下，这是 *React* 做的吗？还是 *React DOM*？

更新 DOM 听起来像是 React DOM 的职责一样。但是我们调用的是 `this.setState()` ，这并不来自于 React DOM。并且，我们的基础类  `React.Component` 是在 React 内部定义的。

所以，在 `React.Component` 内的 `setState()` 是如何更新 DOM 的呢？

**免责声明：就像这个博客中的[其他](/why-do-react-elements-have-typeof-property/)[大多数](/how-does-react-tell-a-class-from-a-function/)[文章](/why-do-we-write-super-props/)一样，使用 React 进行生产并不 *需要* 了解这些内在的原理。这篇文章是为了那些想要了解幕后发生了什么的人写的。你完全可以选择看或不看！**

---

我们可能会认为 `React.Component` 类中包含了 DOM 的更新逻辑。

但如果是那种情况，`this.setState()` 是如何在其他环境中工作的呢？例如，在 React Native 的应用中，组件依旧会继承 `React.Component`。这些组件也像上文所说的那样调用 `this.setState()`，但 React Native 使用的是 Android 和 iOS 原生的视图而不是DOM。

你可能也熟悉 React Test Renderer 或 Shallow Renderer。这两种测试策略都允许你渲染普通组件，并在他们内部调用 `this.setState()`。但他们都不会使用DOM。

如果你使用类似 [React ART](https://github.com/facebook/react/tree/master/packages/react-art) 的渲染器，你可能也会知道在页面中可以使用不只一个渲染器。(例如， ART 组件在 React DOM 树之内工作。)这使得全局标识或变量无法维持。

因此** `React.Component` 将状态更新委托给相应平台的相关代码来处理。**在我们了解这是怎么发生的之前，让我们深入到包是怎么拆分的以及为什么要这么做当中。

---

有一个常见的误解，就是 React “引擎”存在于 `react` 包中。这是不对的。

实际上，自从 [React 0.14 拆分包](https://reactjs.org/blog/2015/07/03/react-v0.14-beta-1.html#two-packages)之后，`react` 包仅有意地导出用于 *定义* 组件的 API。大部分 React 的 *实现* 都存在于“渲染器”中。

例如 `react-dom`，`react-dom/server`，`react-native`，`react-test-renderer`，`react-art` 就是一些渲染器。（你也可以[创建你自己的渲染器](https://github.com/facebook/react/blob/master/packages/react-reconciler/README.md#practical-examples)。）

这就是无论你选择哪一个平台，`react` 都可用的原因。它的所有导出，例如 `React.Component`，`React.createElement`，`React.Children` 与（最终的） [Hooks](https://reactjs.org/docs/hooks-intro.html)，都是独立于目标平台的。无论你运行 React DOM，React DOM Server，还是 React Native，你的组件都可以用相同的方式导入和使用。

相反，那些渲染器的包则导出对应平台的 API，就像 `ReactDOM.render()` 这样，允许你挂在一个 React 的层次结构到 DOM 节点中。每个渲染器都会提供一个类似的API。在理想状况下，大多数 *组件* 不需要从渲染器中导入任何内容。这使它们更加轻便。

**大多数人都想象 React“引擎”是包括在每个独立的渲染器中的。**很多渲染器都包含了同一份代码——我们称之为 [“协调程序”](https://github.com/facebook/react/tree/master/packages/react-reconciler)。[构建步骤](https://reactjs.org/blog/2017/12/15/improving-the-repository-infrastructure.html#migrating-to-google-closure-compiler)会将协调程序的代码与渲染器的代码整合成一个高度优化的捆绑包来获得更高的性能。（复制代码对包的体积控制不是很好，当绝大多数的 React 用户一次只需要一个渲染器，例如 `react-dom`。）

这里要注意的是， `react` 包只允许你 *使用* React 的特性，但是不知道他们是 *怎样* 被实现的。渲染器包(如 `react-dom`, `react-native` 等)提供了 React 的特性和对应平台相关逻辑的实现。其中一些代码是共享的（“协调程序”），但这是单个渲染器的实现细节。

---

现在我们知道为什么 `react`和 `react-dom` 包 *都* 要为了新的特性进行更新。例如，React 16.3 加入了 Context API，`React.createContext()` 方法就在 React 包中被导出。

但实际上 `React.createContext()` 并没有 *实现* context 的特性。例如，在 React DOM 和 React DOM Server 之间就需要不同的实现方式。因此 `createContext()` 只会返回一些简单的对象：

```js
// A bit simplified
function createContext(defaultValue) {
  let context = {
    _currentValue: defaultValue,
    Provider: null,
    Consumer: null
  };
  context.Provider = {
    $$typeof: Symbol.for('react.provider'),
    _context: context
  };
  context.Consumer = {
    $$typeof: Symbol.for('react.context'),
    _context: context,
  };
  return context;
}
```

当你在代码中使用 `<MyContext.Provider>` 或 `<MyContext.Consumer>` 时，是由 *渲染器* 决定如何处理它们。React DOM 会以一种方式来跟踪 context 的值，而 React DOM Server 则可能会用另一种方式。

**因此如果你更新了 `react` 到 16.3 之后的版本，但没有更新 `react-dom`，那么你会使用到一个不支持 `Provider` 和 `Consumer` 类型的渲染器。**这就是为什么旧版本的 `react-dom` 会[告诉你这些类型无效](https://stackoverflow.com/a/49677020/458193)。

在 React Native 中也会有同样的警告。但是，与 React DOM 不同的是，React 的版本发布不会立即“强制”使 React Native 进行版本发布。他们有独立的版本发布时间表。更新后的渲染器代码会在几周后[单独同步](https://github.com/facebook/react-native/commits/master/Libraries/Renderer/oss)到 React Native 的代码库中。这就是 React Native 和 React DOM 在不同时间获得新功能的原因。

---

好了，现在我们已经知道 `react` 包中并不包含那些有趣的东西，并且那些用于实现的代码存放在例如 `react-dom`，`react-native` 的渲染器中，等等。但是这并没有回答我们的问题。在 `React.Component` 中的 `setState()` 应该如何正确地与渲染器“沟通”呢？

**答案就是，每一个渲染器都在创建类时设置了一个特殊制字段。**这个字段被称作 `updater`。这不是 *你* 能设置的——相反，这是 React DOM，React DOM Server 或 React Native 在创建类的示例后立即设置的：


```js{4,9,14}
// Inside React DOM
const inst = new YourComponent();
inst.props = props;
inst.updater = ReactDOMUpdater;

// Inside React DOM Server
const inst = new YourComponent();
inst.props = props;
inst.updater = ReactDOMServerUpdater;

// Inside React Native
const inst = new YourComponent();
inst.props = props;
inst.updater = ReactNativeUpdater;
```

注意 [`setState` 在 `React.Component` 中的实现方式](https://github.com/facebook/react/blob/ce43a8cd07c355647922480977b46713bd51883e/packages/react/src/ReactBaseClasses.js#L58-L67)，它所做的就是将工作委托给创建此组件实例的渲染器：

```js
// A bit simplified
setState(partialState, callback) {
  // Use the `updater` field to talk back to the renderer!
  this.updater.enqueueSetState(this, partialState, callback);
}
```

React DOM Server [可能想要](https://github.com/facebook/react/blob/ce43a8cd07c355647922480977b46713bd51883e/packages/react-dom/src/server/ReactPartialRenderer.js#L442-L448) 忽略状态更新并且向你发送警告，而 React DOM 和 React Native 则会让他们的协调程序副本来[处理](https://github.com/facebook/react/blob/ce43a8cd07c355647922480977b46713bd51883e/packages/react-reconciler/src/ReactFiberClassComponent.js#L190-L207)。

这就是尽管 `this.setState()` 是在 React 包中定义的，也能更新 DOM 的原因。它会读取由React DOM 设置的 `this.updater`，并且让 React DOM 的进行调度和处理更新。