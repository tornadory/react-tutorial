# react-redux
#### react-redux是什么？
在redux中提到过，光有redux还不够，不能和react结合起来，react-redux相当于一个桥梁，才能让两边相通。
通俗点说，单独的react组件相当于一个"傻瓜"（拿不到store数据）组件，只有经过react-redux提供provider和connect连接的组件，就会变成"聪明"（可以拿到store数据）组件。

### API
`<Provider>`   

属性：
* store  全局唯一的store对象
* children  被provider组件包裹的子组件

Provider组件可以让所有经过connect()的组件都能拿到store对象，一般会把应用的根组件包在Provider下。  
没有provider之前的写法：
```javascript
import React from 'react'
import ReactDOM from 'react-dom'
import App from 'App'
  
ReactDOM.render(
  <App />, // App是根组件，APP下的子组件都是拿不到store的
  document.getElementById('root')
)
```
经过provider之后的写法：
```javascript
import React from 'react'
import ReactDOM from 'react-dom'
import { createStore } from 'redux'
import { Provider } from 'react-redux'
import App from './App'
import reducers from './reducers'
  
const store = createStore(reducers)
  
ReactDOM.render(
  <Provider store={store}>   // 经过Provider组件包装后的App根组件，下面的子组件都能通过context拿到store对象。
    <App />
  </Provider>,
  document.getElementById('root')
)
```
子组件通过context拿到store，看起来像这样：
```javascript
import React, { Component }from 'react'
  
class App extends Component {
  render() {
    const { store } = this.context
    const state = store.getState()  // store中的整个state对象： { userName: 'Super', 'age': 18 }
    return (
      <div>{`My name is ${state.userName}, ${state.age} years old.`}</div>  //My name is Super, 18 years old.
    )
  }
}
```
为什么经过provider之后，组件能通过context拿到store对象，这是因为provider把store绑在了context上，父组件通过getChildContext传递属性，子组件通过this.context.xxx访问父组件的属性。context是一个全局对象，尽量避免使用，如需要向子组件传递属性时，组件层级浅时可以通过props传递，组件层级深时，可以通过store全局对象来共享。
```javascript
class Provider extends Component {
  // 子组件可以拿到父组件写在getChildContext其中的返回值属性，父组件其下的所有子组件都能访问的到，所以是个全局对象，应该尽量避免使用。
    getChildContext() {
      return { [storeKey]: this[storeKey], [subscriptionKey]: null }
    }
  
    constructor(props, context) {
      super(props, context)
      this[storeKey] = props.store;
    }
  
    render() {
      return Children.only(this.props.children)
    }
}
```  
      
`connect(mapStateToProps,mapDispatchToProps,mergeProps,options)(WrappedComponent)`  

参数：
* mapStateToProps
* mapDispatchToProps
* mergeProps
* options
* WrappedComponent 与connect连接的组件

返回值：一个新的已与Redux store连接的组件类。
  
参数看起来很多很绕，connect是一个调用两次的高阶函数，第一次接收一些定制的参数，返回一个接收与store连接的组件参数，最终返回一个新的已与store连接的组件类。  

`mapStateToProps(state, ownProps)`  
可选参数，是一个函数，返回一个对象，如果不传，默认是`const defaultMapStateToProps = state => ({})`，如果传了，组件将会监听store的变化。任何时候，只要store发生改变，mapStateToProps函数就会被调用。  
state参数，就是整个大的store.getState()全局对象，ownProps是组件自身的props，一般不传，如果传了，store同样会监听ownProps，组件自身props发生改变也会重新调用mapStateToProps函数。
从名字可以看出，这个函数的作用是把state映射到组件的props上，从而可以让你在组件中通过`this.props.xxx`访问到store数据，当然你也可以取别的函数名，只是这个名字比较语义化。
 
`mapDispatchToProps(dispatch, ownProps)`  
可选参数，可以是个函数，也可以是个对象，一般情况都是函数，返回一个对象。如果不传，默认是`const defaultMapDispatchToProps = dispatch => ({ dispatch })`，即使不传，默认组件可以拿到this.props.dispatch。
dispatch参数，就是store.dispatch，ownProps是组件自身的props，一般也不需要传。如果传了ownProps，组件将会监听props的变化，只要组件接收到新props，mapDispatchToProps也会被调用。  
这个函数的作用是把action经过dispatch包装，然后map到组件的props上，之前说过action都需要dispatch才能被调用，就是在这里去注入dispatch。在组件中通过`this.props.xxx`调用action。

`mergeProps(stateProps, dispatchProps, ownProps)`  
可选参数，经过connect之后的组件中的props来源有三个地方，一个是mapStateToProps，一个是mapDispatchToProps，一个是组件自身的ownProps，mergeProps这个方法三个参数也分别对应，你可以自由的决定组织最终的props是什么样的，哪些舍弃，哪些合并，其实用处不大，一般都在mapStateToProps和mapDispatchToProps中map好就行了。  
mergeProps不传的话，默认返回`Object.assign({}, stateProps, dispatchProps, ownProps)`的结果。

`options`  
可选参数，是个对象。一般不常用，如果指定这个对象，可以定制一些connect的行为：
* pure：如果为true，connect将执行shouldComponentUpdate进行浅对比mergeProps的结果，避免一些不必要的更新，默认值为true。
* withRef： 如果为true，connect保存一个对被包装组件实例的引用，该引用通过getWrappedInstance()方法获得。默认值为false。

给个connect的例子：
```javascript
import React, { Component } from 'react'
import { connect } from 'react-redux'
  
class App extends Component {
  render() {
    return (
      <div onClick={() => this.props.changeUserName()}>
      {this.props.userName}
      </div>
    )
  }
}
  
function mapStateToProps(state) {
  return {
    userName: state.userName
  }
}
  
function mapDispatchToProps(dispatch) {
  return {
    changeUserName: () => dispatch({ type: 'CHANGE_USERNAME', payload: 'Super' })
  }
}
  
export default connect(mapStateToProps, mapDispatchToProps)(App)
  
//  一般情况下都传两个参数,像上面这样
// 当然也可以只传一个，或者一个都不传，像这样：
// connect(mapStateToProps)(App)  mapStateToProps，直接省略mapDispatchToProps参数
// connect(null, mapDispatchToProps)(App)  只mapDispatchToProps, 省略mapStateToProps参数，由于是第一个参数，所以省略时直接传null或者undefined即可。
// connect()(App) 当然也可以两个都不传，不传mapDispatchToProps会默认给组件注入dispatch。
```

#### 至此，react-redux的相关介绍及使用也差不多了，在redux目录下有一个小demo，里面包含了react-redux，感兴趣的同学可以看下。



