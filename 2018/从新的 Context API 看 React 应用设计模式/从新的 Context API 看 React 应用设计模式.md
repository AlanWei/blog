# 从新的 Context API 看 React 应用设计模式
在即将发布的 React v16.3.0 中，React 引入了新的声明式的，可透传 props 的 [Context API](https://link.zhihu.com/?target=https%3A//github.com/facebook/react/pull/11818)，对于新的 Context API 还不太了解朋友可以看一下笔者之前的一个[回答](https://www.zhihu.com/question/267168180/answer/319754359)。

受益于这次改动，React 开发者终于拥有了一个官方提供的安全稳定的 global store，子组件跨层级获取父组件数据及后续更新都不再成为一个问题。这让我们不禁开始思考，相较于  Redux 等其他第三方的数据（状态）管理工具，使用 Context 这种 vanilla React 支持的方式去管理全局数据是不是一个更好的选择呢？

## Context 版本的 Redux
在 react + redux 已经成为开始一个 React 项目标配的今天，我们似乎已经忘记了 react 本身其实是可以使用 state 和 props 来管理数据的。甚至对于市面上大部分的应用来说，对于 redux 的不正确使用实际上增加了应用的复杂度及代码量。

### Vanilla React Global Store
以 react 为例，借助新的 Context API，我们可以轻松地将某个组件中的数据同步到其他任意一个需要同样数据的组件中（theme，color），也可以将触发复杂异步操作的 handler 注入到需要的组件中（changeContextAsync），以达到代码复用的目的。

```javascript
import React from "react";
import { render } from "react-dom";

const initialState = {
  theme: "dark",
  color: "blue"
};

const GlobalStoreContext = React.createContext({
  ...initialState
});

class GlobalStoreContextProvider extends React.Component {
  // initialState
  state = {
    ...initialState
  };

  // reducer
  handleContextChange = action => {
    switch (action.type) {
      case "UPDATE_THEME":
        return this.setState({
          theme: action.theme
        });
      case "UPDATE_COLOR":
        return this.setState({
          color: action.color
        });
      case "UPDATE_THEME_THEN_COLOR":
        return new Promise(resolve => {
          resolve(action.theme);
        })
          .then(theme => {
            this.setState({
              theme
            });
            return action.color;
          })
          .then(color => {
            this.setState({
              color
            });
          });
      default:
        return;
    }
  };

  render() {
    return (
      <GlobalStoreContext.Provider
        value={{
          dispatch: this.handleContextChange,
          theme: this.state.theme,
          color: this.state.color
        }}
      >
        {this.props.children}
      </GlobalStoreContext.Provider>
    );
  }
}

const SubComponent = props => (
  <div>
    {/* action */}
    <button
      onClick={props.dispatch({
        type: "UPDATE_THEME",
        theme: "light"
      })}
    >
      change theme
    </button>
    <div>{props.theme}</div>
    {/* action */}
    <button
      onClick={props.dispatch({
        type: "UPDATE_COLOR",
        color: "red"
      })}
    >
      change color
    </button>
    <div>{props.color}</div>
    {/* action */}
    <button
      onClick={props.dispatch({
        type: "UPDATE_COLOR",
        color: "red"
      })}
    >
      change theme then color
    </button>
  </div>
);

class App extends React.Component {
  render() {
    return (
      <GlobalStoreContextProvider>
        <GlobalStoreContext.Consumer>
          {context => (
            <SubComponent
              theme={context.theme}
              color={context.color}
              dispatch={context.dispatch}
            />
          )}
        </GlobalStoreContext.Consumer>
      </GlobalStoreContextProvider>
    );
  }
}

render(<App />, document.getElementById("root"));
```

## Redux 的实际用途
相较于上面 Context 版本的 Redux，原生版本的 Redux 的核心竞争力并不在于数据更新（获取），action 分发等大部分人经常用到的功能，而是它的**中间件机制**。

在 Context 版本中，一个用户行为（click）会直接触发相应的 handler 函数去更新数据，而在原生版本的 redux 中，因为整个 action dispatch cycle 的存在，开发者可以在 dispatch action 前后，中心化地利用中间件机制去更好地跟踪整个过程，如我们常说的 action logger 及 time travel 这些功能，其本质依赖的就是 redux 的中间件机制。

换句话说，我们也可以让 Context 版本中的所有 handler 在更新 state 之前都先通过一个统一的函数
 
## 展示型组件（Presentational） vs. 容器型组件（Container）
时间回到 2015 年，那时的 React 刚刚发布了 0.13 版本，Redux 也还没有成为

## Redux 带来的问题


## Context + Redux = 更好的 React 应用设计模式