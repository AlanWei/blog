# 从新的 Context API 看应用状态管理
在谈新的 Context API 之前，我们有必要先来看一下现在的 [Context API](https://reactjs.org/docs/context.html) 存在着哪些问题。

Context 作为一个实验性质的 API，直到 React v16.3.0 版本前都一直不被官方所提倡去使用，其主要原因就是因为在子组件中使用 Context 会破坏 React 应用的**分形**架构。

这里的分形架构指的是从理想的 React 应用的根组件树中抽取的任意一部分都仍是一个可以直接运行的子组件树。在这个子组件树之上再包一层，就可以将它无缝地移植到任意一个其他的根组件树中。

但如果根组件树中有任意一个组件使用了支持 `props` 透传的 Context API，那么如果把包含了这个组件的子组件树单独拿出来，因为缺少了提供 Context 值的根组件树，这时的这个子组件树是无法直接运行的。

另一方面，虽然 React 官方不推崇使用 Context API，我们在日常工作中却每天都在使用着这个实验性质的 API。或者说虽然开发者可能在自己的应用中并没有直接调用过，但应用本身依赖的 [react-redux](https://github.com/reactjs/react-redux/blob/76dd7faa90981dd2f9efa76f3e2f26ecf2c12cf7/src/components/connectAdvanced.js#L136-L143) 或 [mobx-react](https://github.com/mobxjs/mobx-react/blob/dc249910c74c1b2e988a879be07f10aeaea90936/src/Provider.js#L19-L34) 等状态管理库之所以能够实现在任意组件中访问全局 store 这一功能，其基础依赖的就是 Context API。

## 现有 Context API 的致命缺陷
现有的原生 Context API 存在着一个致命的问题，那就是在 Context 值更新后，顶层组件向目标组件 `props` 透传的过程中，如果中间某个组件的 `shouldComponentUpdate` 函数返回了 `false`，因为无法再继续触发底层组件的 rerender，新的 Context 值将无法到达目标组件。这样的不确定性对于目标组件来说是完全不可控的，也就是说目标组件无法保证自己每一次都可以接收到更新后的 Context 值。

## 现有 Context API 的用法
这是 React 官网上一段介绍现在 Context API 用法的一段代码：

```javascript
import PropTypes from 'prop-types';

class Button extends React.Component {
  render() {
    return (
      <button style={{background: this.context.color}}>
        {this.props.children}
      </button>
    );
  }
}

Button.contextTypes = {
  color: PropTypes.string
};

class Message extends React.Component {
  render() {
    return (
      <div>
        {this.props.text} <Button>Delete</Button>
      </div>
    );
  }
}

class MessageList extends React.Component {
  getChildContext() {
    return {color: "purple"};
  }

  render() {
    const children = this.props.messages.map((message) =>
      <Message text={message.text} />
    );
    return <div>{children}</div>;
  }
}

MessageList.childContextTypes = {
  color: PropTypes.string
};
```

我们重点来看这两行代码：

```javascript
// SubComponent
<button style={{background: this.context.color}}>

// RootComponent
getChildContext() {
  return {color: "purple"};
}
```

一句话来说，这样使用 Context API 的方法非常不 React，目标组件中的 `this.context` 非常 magic，顶层组件中 `getChildContext()` 也与 React 本身所推崇的声明式写法背道而驰。

## 新的 Context API
[新的 Context API](https://github.com/facebook/react/pull/11818) 采用声明式的写法，并且可以透过 `shouldComponentUpdate` 返回 `false` 的组件继续向下传播，以保证目标组件一定可以接收到顶层组件 Context 值的更新，一举解决了现有 Context API 的两大弊端，也终于成为了 React 中的第一级（first-class） API。

让我们来看一个 Demo：

```javascript
import React from "react";
import { render } from "react-dom";

const ThemeContext = React.createContext();

class ThemeProvider extends React.Component {
  state = {
    theme: "dark",
    color: "blue"
  };

  changeTheme = theme => {
    this.setState({
      theme
    });
  };

  changeColor = color => {
    this.setState({
      color
    });
  };

  render() {
    return (
      <ThemeContext.Provider
        value={{
          theme: this.state.theme,
          color: this.state.color,
          changeColor: this.changeColor
        }}
      >
        <button onClick={() => this.changeTheme("light")}>change theme</button>
        {this.props.children}
      </ThemeContext.Provider>
    );
  }
}

const SubComponent = props => (
  <div>
    <div>{props.theme}</div>
    <button onClick={() => props.changeColor("red")}>change color</button>
    <div>{props.color}</div>
  </div>
);

class App extends React.Component {
  render() {
    return (
      <ThemeProvider>
        <ThemeContext.Consumer>
          {context => (
            <SubComponent
              theme={context.theme}
              color={context.color}
              changeColor={context.changeColor}
            />
          )}
        </ThemeContext.Consumer>
      </ThemeProvider>
    );
  }
}

render(<App />, document.getElementById("app"));
```

新的 Context API 分为三个组成部分：

* `React.createContext` 用于初始化一个 Context。
* `XXXContext.Provider` 作为顶层组件接收一个名为 `value` 的 `prop`，可以接收任意需要被放入 Context 中的字符串，数字，甚至是函数。
* `XXXContext.Consumer` 作为目标组件可以出现在组件树的任意位置（在 `Provider` 之后），接收 children prop，这里的 children 必须是一个函数（`context => ()`）用来接收从顶层传来的 Context。

## Context 与 Redux 不得不说的故事
在拥有了透过 `shouldComponentUpdate` 返回 `false` 组件的能力后，我们可以使用新的 Context API 来写一个非常简单的 Redux：

```javascript
import React from "react";
import { render } from "react-dom";

const initialState = {
  theme: "dark",
  color: "blue"
};

const GlobalStore = React.createContext();

class GlobalStoreProvider extends React.Component {
  render() {
    return (
      <GlobalStore.Provider value={{ ...initialState }}>
        {this.props.children}
      </GlobalStore.Provider>
    );
  }
}

class App extends React.Component {
  render() {
    return (
      <GlobalStoreProvider>
        <GlobalStore.Consumer>
          {context => (
            <div>
              <div>{context.theme}</div>
              <div>{context.color}</div>
            </div>
          )}
        </GlobalStore.Consumer>
      </GlobalStoreProvider>
    );
  }
}

render(<App />, document.getElementById("root"));
```

这让我们不禁想起了 Dan Abramov 2 年前的一篇文章 [You Might Not Need Redux](https://medium.com/@dan_abramov/you-might-not-need-redux-be46360cf367)。如果说在现有 Context API 的基础上我们还需要 `react-redux` 的帮助去克服无法穿透 `shouldComponentUpdate` 返回 `false` 组件这个障碍的话，在新的 Context API 的语境中就真得不需要再将 `redux` 与 `react-redux` 作为项目开始时就必须安装的依赖了，如果我们只需要 `props` 透传这一个特性的话。

甚至在数据管理方面，新的 Context 还可以做得更好。新的 Context API 不受单一 store 的限制，每一个 Context 都相当于 store 中的一个分支，我们可以创建多个 Context 来管理不同类型的数据，相应的在使用时也可以只为目标组件上包上需要的 Context Provider。

最后，受益于新的 Context API 的声明式写法，我们终于可以抛开 Connect 轻松地写符合分形要求的业务组件了，如上面代码中的 `SubComponent`：

```javascript
const SubComponent = props => (
  <div>
    <div>{props.theme}</div>
    <button onClick={() => props.changeColor("red")}>change color</button>
    <div>{props.color}</div>
  </div>
);

const SubComponentWithContext = props => (
  <ThemeContext.Consumer>
    {context => (
      <SubComponent
        theme={context.theme}
        color={context.color}
        changeColor={context.changeColor}
      />
    )}
  </ThemeContext.Consumer>
)
```

相较于 Redux 的写法，整体代码直观了许多，在后期组件层级调整时，代码的整体可复用性也提升了许多。