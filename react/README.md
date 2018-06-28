# 用户API分析（提取重点接口）
首先我们看一下React的基本使用方法

```
//创建一个React组件
var ExampleApplication = React.createClass({
	render: function() {
	  var elapsed = Math.round(this.props.elapsed  / 100);
	  var seconds = elapsed / 10 + (elapsed % 10 ? '' : '.0' );
	  var message =
	    'React has been successfully running for ' + seconds + ' seconds.';
	
	  return `<h1>{message}</h1>`;
	}
});

var start = new Date().getTime();

setInterval(function() {
	//将React组件渲染成真正的dom元素
	React.renderComponent(
	  <ExampleApplication elapsed={new Date().getTime() - start} />,
	  document.getElementById('container')
	);
}, 50);
```
在这里我们可以看到用户API的两个主要方法

* React.createClass
* React.renderComponent

还有facebook自主开发的js扩展语法jsx

在这里我们主要学习框架的主要思想，jsx更像是一种语言模板我们就不多讲了

# 代码分析

## 1. 入口

React的入口代码可以看出大概分为两个部分

* 注入 inject
* 导出 expoxt

### 注入
```
ReactDefaultInjection.inject();
```
注入包括

```
//用于解决DOM层次结构和插件顺序的注入模块
EventPluginHub.injection.injectEventPluginOrder(DefaultEventPluginOrder);
EventPluginHub.injection.injectInstanceHandle(ReactInstanceHandles);
```

```
//默认注入的两个重要模块
EventPluginHub.injection.injectEventPluginsByName({
	'SimpleEventPlugin': SimpleEventPlugin,
	'EnterLeaveEventPlugin': EnterLeaveEventPlugin
});
```

```
//将'form'元素覆盖为react标准组件，因为IE8不会冒泡或捕获提交到顶层。为了完成这项工作，我们需要在这里注入它。
ReactDOM.injection.injectComponentClasses({
    form: ReactDOMForm
});
```

### 导出React模块

```
var React = {
  DOM: ReactDOM,
  initializeTouchEvents: function(shouldUseTouch) {
    ReactMount.useTouchEvents = shouldUseTouch;
  },
  autoBind: ReactCompositeComponent.autoBind,
  createClass: ReactCompositeComponent.createClass,  // important
  createComponentRenderer: ReactMount.createComponentRenderer,
  constructAndRenderComponent: ReactMount.constructAndRenderComponent,
  constructAndRenderComponentByID: ReactMount.constructAndRenderComponentByID,
  renderComponent: ReactMount.renderComponent,  // important
  unmountAndReleaseReactRootNode: ReactMount.unmountAndReleaseReactRootNode,
  isValidComponent: ReactComponent.isValidComponent
};
```
在导出代码中我们不难发现在开始我们提取出的两个重要方法，现在我们从第一个方法开始看

#### React.createClass
我们转到 ReactCompositeComponent.createClass 看一下

```
createClass: function(spec) {
	var Constructor = function() {};
	Constructor.prototype = new ReactCompositeComponentBase();
	Constructor.prototype.constructor = Constructor;
	mixSpecIntoComponent(Constructor, spec);
	invariant(
		Constructor.prototype.render,
		'createClass(...): Class specification must implement a `render` method.'
	);

	var ConvenienceConstructor = function(props, children) {
		var instance = new Constructor();
		instance.construct.apply(instance, arguments);
		return instance;
	};
	ConvenienceConstructor.componentConstructor = Constructor;
	ConvenienceConstructor.originalSpec = spec;
	return ConvenienceConstructor;
}
```
1. 创建类Constructor继承ReactCompositeComponentBase然后进行参数混合（mix spec into component）的操作并判断是否有render，如果没有就报错
2. 创建ConvenienceConstructor并返回，在ConvenienceConstructor实例化的时候进行Construtor的实例化赋值给instance并执行instance的construct方法

这里一共有几个问题

1. ReactCompositeComponentBase是什么？
2. instance的construct方法在哪里又做了什么？

初步思路，instance是ReactCompositeComponentBase的一个实例，那construct方法一定也存在于其中，那我们继续跳转到ReactCompositeComponentBase看一下

```
var ReactCompositeComponentBase = function() {};
mixInto(ReactCompositeComponentBase, ReactComponent.Mixin);
mixInto(ReactCompositeComponentBase, ReactOwner.Mixin);
mixInto(ReactCompositeComponentBase, ReactPropTransferer.Mixin);
mixInto(ReactCompositeComponentBase, ReactCompositeComponentMixin);
```

ReactCompositeComponentBase混合了四个东西，打开ReactComponent.Mixin，这里有行注释大概解释了Mixin的作用

```
* Returns the DOM node rendered by this component.
//返回由该组件呈现的DOM节点
```

我们大概看了一下Mixin里的内容，惊喜万分，发现了construct方法，还有一些其他的mountComponent,unmountComponent方法，还有一行醒目的注释

```
* Base constructor for all React component.
//所有React组件的基本构造函数

* Subclasses that override this method should make sure to invoke
* `ReactComponent.Mixin.construct.call(this, ...)`.
//子类覆盖此方法应该执行一下`ReactComponent.Mixin.construct.call(this, ...)`，换句话讲，在标准的面向对象语言中的子类在构造函数中执行super()
```

Mixin中的方法太多我们不用太关心，不影响我们理解框架核心思想，继续往后看到ReactCompositeComponentMixin，我们会发现这里的方法比较眼熟，这里也有construct, mountComponent, unmountComponent，大概明白了这里继承（覆盖）了ReactComponent.Mixin中的一些方法，那instance里的construct应该是这里的construct没错了，construct的作用大概想想都知道是初始化了就不多说了，有兴趣可以自主查看一下construct的实现，里面有备注也比较好理解

那到这里，craeteClass的原理我们清楚了，就是创建一个React组件（ConvenienceConstructor类），中间混合了很多方法，然后ConvenienceConstructor实例化会执行construct方法，construct方法里做了什么后续我会完善，记个todo#1

#### React.renderComponent
顾名思义，渲染组件，我们来看一下他的内部实现
我们转到 ReactMount.renderComponent看一下

先看一下注释

```
* Renders a React component into the DOM in the supplied `container`.
//将React组件渲染到`container`容器中
*
* If the React component was previously rendered into `container`, this will
* perform an update on it and only mutate the DOM as necessary to reflect the
* latest React component.
//如果之前已经将组件渲染到容器了，则将对其进行更新，并仅在必要时对DOM进行突变以反映最新的组件。
```

再看代码就很好理解了

```
renderComponent: function(nextComponent, container) {
	//获取组件ID（框架内部机制定义的唯一ID），判断组件是否已经渲染，如果有就更新
	var prevComponent = instanceByReactRootID[getReactRootID(container)];
	if (prevComponent) {
		var nextProps = nextComponent.props;
		ReactMount.scrollMonitor(container, function() {
			prevComponent.replaceProps(nextProps);
		});
	return prevComponent;
	}
	
	//这里是初次渲染的逻辑
	ReactMount.prepareTopLevelEvents(ReactEventTopLevelCallback);

	var reactRootID = ReactMount.registerContainer(container);
	instanceByReactRootID[reactRootID] = nextComponent;
	nextComponent.mountComponentIntoNode(reactRootID, container);
	return nextComponent;
},
```
我们带着以下两个问题具体看一下？

1. 组件渲染的逻辑是什么
2. 组件更新的逻辑是什么

#### 组件渲染部分 render

```
//注册顶级事件监听代理
//具体作用后续更新#2
ReactMount.prepareTopLevelEvents(ReactEventTopLevelCallback);

var reactRootID = ReactMount.registerContainer(container);
instanceByReactRootID[reactRootID] = nextComponent;
nextComponent.mountComponentIntoNode(reactRootID, container);
return nextComponent;
```

#1 construct