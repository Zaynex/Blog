[2018-02-09————待定]

## React-router
需要学习的几个部分：
1. context 传递/new Context api 
2. history 历史的页面数据
3. path url 匹配
4. 服务端渲染

### Router 架构
其实传输的方式和 redux 的 store 是一样的，本质上也是用了 context.
```
import createHistory from 'history/createBrowserHistory'
const history = createHistory()

class App extends Component {
  static propTypes = {
    store: PropTypes.object
  }
  render () {
    const { store } = this.props
    return (
      <Provider store={store}>
        <Router history={history}>
          <Switch>
            <Route path="/" component={Component} />
            <Redirect to="/" />
          </Switch>
        </Router>
      </Provider>
    )
  }
}
```

将 router 和 history 等数据合并
```
class Router extends React.Component {
	static childContextTypes = {
		router: PropTypes.object.isRequired
	}

	static contextTypes = {
		router: PropTypes.object
	}
	getChildContext() {
		return {
			router: {
				...this.context.router,
				history: this.props.history
			}
		}
	}
}
```

### Route 架构
Route 承载了渲染具体某个组件的职责。我们只要在Route 通过 context 拿到所有数据，即当面页面的路由，再根据传入 path 匹配就知道该渲染哪个组件了。

```
import matchPath from "./matchPath";
class Route extends React.Component {
	computeMatch(
    { computedMatch, location, path, strict, exact, sensitive },
    router
  ) {
    if (computedMatch) return computedMatch; // <Switch> already computed the match for us

    invariant(
      router,
      "You should not use <Route> or withRouter() outside a <Router>"
    );

    const { route } = router;
    const pathname = (location || route.location).pathname;

    return path
      ? matchPath(pathname, { path, strict, exact, sensitive })
      : route.match;
  }
	state = {
		match: this.computeMatch(this.props, this.context.router)
	}
	componentWillReceiveProps(nextProps, nextContext) {
		this.setState({
      match: this.computeMatch(nextProps, nextContext.router)
    });
	}
  render() {
	  const { match } = this.state;
	  const { children, component, render } = this.props;
	  const { history, route, staticContext } = this.context.router;
	  const location = this.props.location || route.location;
	  const props = { match, location, history, staticContext };

	  if (component) return match ? React.createElement(component, props) : null;

	  if (render) return match ? render(props) : null;

	  if (typeof children === "function") return children(props);

	  if (children && !isEmptyChildren(children))
	    return React.Children.only(children);

	  return null;
	}
}
```


### matchPath 处理逻辑
主要依赖第三方库生成相应的正则去匹配
具体库的地址：`https://github.com/pillarjs/path-to-regexp`


### history 
主要是分三块。一般用的比较多的是 `createBrowserHistory`。 使用 HTML5 新的API history 进行路由的切换。
createMemoryHistory 是针对 DOM 无关的第三方平台使用的。
createHashHistory 针对老浏览器，可以不关注。
