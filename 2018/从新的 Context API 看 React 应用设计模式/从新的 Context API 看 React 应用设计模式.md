# 从新的 Context API 看 React 应用设计模式
在即将发布的 React v16.3.0 中，React 引入了新的声明式的，可透传 props 的 [Context API](https://github.com/facebook/react/pull/11818)，对于新版 Context API 还不太了解朋友可以看一下笔者之前的一个[回答](https://www.zhihu.com/question/267168180/answer/319754359)。

受益于这次改动，React 开发者终于拥有了一个官方提供的安全稳定的 global store，子组件跨层级获取父组件数据及后续的更新都不再成为一个问题。这让我们不禁开始思考，相较于 Redux 等其他的第三方数据（状态）管理工具，使用 Context API 这种 vanilla React 支持的方式是不是一个更好的选择呢？

## Context vs. Redux
在 react + redux 已经成为了开始一个 React 项目标配的今天，我们似乎忘记了其实 react 本身是可以使用 state 和 props 来管理数据的，甚至对于目前市面上大部分的应用来说，对 redux 的不正确使用实际上增加了应用整体的复杂度及代码量。

### Vanilla React Global Store
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
      onClick={() =>
        props.dispatch({
          type: "UPDATE_THEME",
          theme: "light"
        })
      }
    >
      change theme
    </button>
    <div>{props.theme}</div>
    {/* action */}
    <button
      onClick={() =>
        props.dispatch({
          type: "UPDATE_COLOR",
          color: "red"
        })
      }
    >
      change color
    </button>
    <div>{props.color}</div>
    {/* action */}
    <button
      onClick={() =>
        props.dispatch({
          type: "UPDATE_THEME_THEN_COLOR",
          theme: "monokai",
          color: "purple"
        })
      }
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

在上面的例子中，我们使用 Context API 实现了一个简单的 redux + react-redux，这证明了在新版 Context API 的支持下，原先 react-redux 帮我们做的一些工作现在我们可以自己来做了。另一方面，对于已经厌倦了整天都在写 action 和 reducer 的朋友们来说，在上面的例子中忽略掉 dispatch，action 等这些 Redux 中的概念，直接调用 React 中常见的 handleXXX 方法来 setState 也是完全没有问题的，可以有效地缓解 Redux 模板代码过多的问题。而对于 React 的初学者来说，更是省去了学习 Redux 及函数式编程相关概念与用法的过程。

### 正确地使用 Redux
从上面 Context 版本的 Redux 中可以看出，如果我们只需要 Redux 来做全局数据源并配合 props 透传使用的话，新版的 Context 可能是一个可以考虑的更简单的替代方案。另一方面，原生版本 Redux 的核心竞争力其实也并不在于此，而是其**中间件机制**以及社区中一系列非常成熟的中间件。

在 Context 版本中，用户行为（click）会直接调用 reducer 去更新数据。而在原生版本的 Redux 中，因为整个 action dispatch cycle 的存在，开发者可以在 dispatch action 前后，中心化地利用中间件机制去更好地跟踪/管理整个过程，如常用的 action logger，time travel 等中间件都受益于此。

### 渐进式地选择数据流工具
#### Context
* 我需要一个全局数据源且其他组件可以直接获取/改变全局数据源中的数据

#### Redux
* 我需要一个全局数据源且其他组件可以直接获取/改变全局数据源中的数据
* 我需要全程跟踪/管理 action 的分发过程/顺序

#### redux-thunk
* 我需要一个全局数据源且其他组件可以直接获取/改变全局数据源中的数据
* 我需要全程跟踪/管理 action 的分发过程/顺序
* 我需要组件对同步或异步的 action 无感，调用异步 action 时不需要显式地传入 dispatch

#### redux-saga
* 我需要一个全局数据源且其他组件可以直接获取/改变全局数据源中的数据
* 我需要全程跟踪/管理 action 的分发过程/顺序
* 我需要组件对同步或异步的 action 无感，调用异步 action 时不需要显式地传入 dispatch
* 我需要声明式地来表述复杂异步数据流（如长流程表单，请求失败后重试等），命令式的 thunk 对于复杂异步数据流的表现力有限

## Presentational vs. Container
时间回到 2015 年，那时 React 刚刚发布了 0.13 版本，Redux 也还没有成为 React 应用的标配，前端开发界讨论的[主题](https://medium.com/@dan_abramov/smart-and-dumb-components-7ca2f9a7c7d0)是 **React 组件的最佳设计模式**，后来大家得出的结论是将所有组件分为 Presentational（展示型） 及 Container（容器型）两类可以极大地提升组件的可复用性。

但后来 Redux 的广泛流行逐渐掩盖了这个非常有价值的结论，开发者们开始习惯性地将所有组件都 connect 到 redux store 上，以方便地获取所需要的数据。

组件与组件之间的层级结构渐渐地只存在于 DOM 层面，大量展示型的组件被 connect 到了 redux store 上，以至于在其他页面想要复用这个组件时，开发者们更倾向于复制粘贴部分代码。最终导致了 redux store 越来越臃肿，应用的数据流并没有因为引入 Redux 而变得清晰，可复用的展示型组件越来越少，应用与应用之间越来越独立，没有人再愿意去思考应用层面的抽象与复用，项目越做越多，收获的却越来越少。

当所有的组件都与数据耦合在一起，视图层与数据层之间的界限也变得越来越模糊，这不仅彻底打破了 React 本身的分形结构，更是造成应用复杂度陡增的罪魁祸首。

## Context + Redux = 更好的 React 应用设计模式
除了更克制地使用 connect，区分展示型与容器型组件之外，受制于现在 Context API，开发者通常也会将主题，语言文件等数据挂在 redux store 的某个分支上。对于这类不常更新，却需要随时可以注入到任意组件的数据，使用新的 Context API 来实现依赖注入显然是一个更好的选择。

```javascript
import React from "react";
import { render } from "react-dom";
import { createStore } from "redux";
import { Provider, connect } from "react-redux";

const ThemeContext = React.createContext("light");
class ThemeProvider extends React.Component {
  state = {
    theme: "light"
  };

  render() {
    return (
      <ThemeContext.Provider value={this.state.theme}>
        {this.props.children}
      </ThemeContext.Provider>
    );
  }
}
const LanguageContext = React.createContext("en");
class LanguageProvider extends React.Component {
  state = {
    laguage: "en"
  };

  render() {
    return (
      <LanguageContext.Provider value={this.state.laguage}>
        {this.props.children}
      </LanguageContext.Provider>
    );
  }
}
const initialState = {
  todos: []
};
const todos = (state, action) => {
  switch (action.type) {
    case "ADD_TODO":
      return {
        todos: state.todos.concat([action.text])
      };
    default:
      return state;
  }
};
function AppProviders({ children }) {
  const store = createStore(todos, initialState);
  return (
    <Provider store={store}>
      <LanguageProvider>
        <ThemeProvider>{children}</ThemeProvider>
      </LanguageProvider>
    </Provider>
  );
}
function ThemeAndLanguageConsumer({ children }) {
  return (
    <LanguageContext.Consumer>
      {language => (
        <ThemeContext.Consumer>
          {theme => children({ language, theme })}
        </ThemeContext.Consumer>
      )}
    </LanguageContext.Consumer>
  );
}

const TodoList = props => (
  <div>
    <div>
      {props.theme} and {props.language}
    </div>
    {props.todos.map((todo, idx) => <div key={idx}>{todo}</div>)}
    <button onClick={props.handleClick}>add todo</button>
  </div>
);

const mapStateToProps = state => ({
  todos: state.todos
});

const mapDispatchToProps = {
  handleClick: () => ({
    type: "ADD_TODO",
    text: "Awesome"
  })
};

const ToDoListContainer = connect(mapStateToProps, mapDispatchToProps)(
  TodoList
);

class App extends React.Component {
  render() {
    return (
      <AppProviders>
        <ThemeAndLanguageConsumer>
          {({ theme, language }) => (
            <ToDoListContainer theme={theme} language={language} />
          )}
        </ThemeAndLanguageConsumer>
      </AppProviders>
    );
  }
}

render(<App />, document.getElementById("root"));
```

在上面的这个完整的例子中，通过组合多个 Context Provider，我们最终得到了一个组合后的 Context Consumer：

```javascript
<ThemeAndLanguageConsumer>
  {({ theme, language }) => (
    <ToDoListContainer theme={theme} language={language} />
  )}
</ThemeAndLanguageConsumer>
```

另一方面，通过分离展示型组件和容器型组件，我们得到了一个纯净的 `TodoList` 组件：

```javascript
const TodoList = props => (
  <div>
    <div>
      {props.theme} and {props.language}
    </div>
    {props.todos.map((todo, idx) => <div key={idx}>{todo}</div>)}
    <button onClick={props.handleClick}>add todo</button>
  </div>
);
```

## 小结
在 React v16.3.0 正式发布后，用 Context 来做依赖注入（theme，intl，buildConfig），用 Redux 来管理数据流，渐进式地根据业务场景选择 redux-thunk，redux-saga 或 redux-observable 来处理复杂异步情况，可能会一种更好的 React 应用设计模式。

选择用什么样的工具从来都不是决定一个开发团队成败的关键，根据业务场景选择恰当的工具，并利用工具反过来约束开发者，最终达到控制整体项目复杂度的目的，才是促进一个开发团队不断提升的核心动力。