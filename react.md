## react source code analysis
---
看之前要先了解官网关于自己实现的介绍

[react code base](https://facebook.github.io/react/contributing/codebase-overview.html)

[implementation notes](https://facebook.github.io/react/contributing/implementation-notes.html)

[reconciliation](https://facebook.github.io/react/docs/reconciliation.html)
react整体项目目录结构browserify前全部平级化了，详细看
[react模块系统](https://leozdgao.me/react-global-module-system/)

###  Part 1 ReactDom.render

我们从React 渲染开始研究

```javascript
class App extends React.Component {
  render() {
    return <div> test</div>;
  }
}
ReactDom.render(<App />, document.querySelector('#id'));
```
文件在 react/src/renderers/dom/stack/client/ReactMount.js
注意render方法其实是接受cb的
```javascript
  /**
   * Renders a React component into the DOM in the supplied `container`.
   * See https://facebook.github.io/react/docs/react-dom.html#render
   *
   * If the React component was previously rendered into `container`, this will
   * perform an update on it and only mutate the DOM as necessary to reflect the
   * latest React component.
   *
   * @param {ReactElement} nextElement Component element to render.
   * @param {DOMElement} container DOM element to render into.
   * @param {?function} callback function triggered on completion
   * @return {ReactComponent} Component instance rendered in `container`.
   */
  render: function(nextElement, container, callback) {
    return ReactMount._renderSubtreeIntoContainer(null, nextElement, container, callback);
  }
```
这里简单地调用了_renderSubtreeIntoContainer，初始化的时候肯定没有parentComponent，只有container也就是我们的挂载点，
nextElement就是我们现在要渲染的Component

```javascript
_renderSubtreeIntoContainer: function(parentComponent, nextElement, container, callback) {
    validateCallback(callback, 'ReactDOM.render');
   
    var nextWrappedElement = React.createElement(
      TopLevelWrapper,
      { child: nextElement }
    );

    var nextContext = getContextForSubtree(parentComponent);
    var prevComponent = getTopLevelWrapperInContainer(container);

    if (prevComponent) {
      var prevWrappedElement = prevComponent._currentElement;
      var prevElement = prevWrappedElement.props.child;
      if (shouldUpdateReactComponent(prevElement, nextElement)) {
        var publicInst = prevComponent._renderedComponent.getPublicInstance();
        var updatedCallback = callback && function() {
          callback.call(publicInst);
        };
        ReactMount._updateRootComponent(
          prevComponent,
          nextWrappedElement,
          nextContext,
          container,
          updatedCallback
        );
        return publicInst;
      } else {
        ReactMount.unmountComponentAtNode(container);
      }
    }

    var reactRootElement = getReactRootElementInContainer(container);
    var containerHasReactMarkup =
      reactRootElement && !!internalGetID(reactRootElement);
    var containerHasNonRootReactChild = hasNonRootReactChild(container);

      if (!containerHasReactMarkup || reactRootElement.nextSibling) {
        var rootElementSibling = reactRootElement;
        while (rootElementSibling) {
          if (internalGetID(rootElementSibling)) {
            warning(
              false,
              'render(): Target node has markup rendered by React, but there ' +
              'are unrelated nodes as well. This is most commonly caused by ' +
              'white-space inserted around server-rendered markup.'
            );
            break;
          }
          rootElementSibling = rootElementSibling.nextSibling;
        }
      }
    }

    var shouldReuseMarkup =
      containerHasReactMarkup &&
      !prevComponent &&
      !containerHasNonRootReactChild;
    var component = ReactMount._renderNewRootComponent(
      nextWrappedElement,
      container,
      shouldReuseMarkup,
      nextContext,
      callback
    )._renderedComponent.getPublicInstance();
    return component;
  },
```

其中
```javascript
var nextWrappedElement = React.createElement(
  TopLevelWrapper,
  { child: nextElement }
);
```
为rootComponent创建react element
```javascript
var prevComponent = getTopLevelWrapperInContainer(container);
```
因为我们是服务端渲染的 getTopLevelWrapperInContainer(container)得出prevComponent为空
```javascript
if (prevComponent) {
  var prevWrappedElement = prevComponent._currentElement;
  var prevElement = prevWrappedElement.props.child;
  if (shouldUpdateReactComponent(prevElement, nextElement)) {
    var publicInst = prevComponent._renderedComponent.getPublicInstance();
    var updatedCallback = callback && function() {
      callback.call(publicInst);
    };
    ReactMount._updateRootComponent(
      prevComponent,
      nextWrappedElement,
      nextContext,
      container,
      updatedCallback
    );
    return publicInst;
  } else {
    ReactMount.unmountComponentAtNode(container);
  }
}
```
如果prevComponent不为空的情况，会判断是否要update component
shouldUpdateReactComponent逻辑为
```javascript
/**
 * Given a `prevElement` and `nextElement`, determines if the existing
 * instance should be updated as opposed to being destroyed or replaced by a new
 * instance. Both arguments are elements. This ensures that this logic can
 * operate on stateless trees without any backing instance.
 *
 * @param {?object} prevElement
 * @param {?object} nextElement
 * @return {boolean} True if the existing instance should be updated.
 * @protected
 */

function shouldUpdateReactComponent(prevElement, nextElement) {
  var prevEmpty = prevElement === null || prevElement === false;
  var nextEmpty = nextElement === null || nextElement === false;
  if (prevEmpty || nextEmpty) {
    return prevEmpty === nextEmpty;
  }

  var prevType = typeof prevElement;
  var nextType = typeof nextElement;
  if (prevType === 'string' || prevType === 'number') {
    return nextType === 'string' || nextType === 'number';
  } else {
    return nextType === 'object' && prevElement.type === nextElement.type && prevElement.key === nextElement.key;
  }
}
```
我猜测TextNode type为string或者number，那个前后都为TextNode 返回为true
CompositeComponent的type为object，且组件类型一直就返回为true

返回true之后拿到PublicInstance，PublicInstance为user-defined component 的实例
判断是否有cb,去执行cb,然后执行ReactMount._updateRootComponent
```javascript
  /**
   * Take a component that's already mounted into the DOM and replace its props
   * @param {ReactComponent} prevComponent component instance already in the DOM
   * @param {ReactElement} nextElement component instance to render
   * @param {DOMElement} container container to render into
   * @param {?function} callback function triggered on completion
   */
  _updateRootComponent: function (prevComponent, nextElement, nextContext, container, callback) {
    ReactMount.scrollMonitor(container, function () {
      ReactUpdateQueue.enqueueElementInternal(prevComponent, nextElement, nextContext);
      if (callback) {
        ReactUpdateQueue.enqueueCallbackInternal(prevComponent, callback);
      }
    });

    return prevComponent;
  }
```
扔给ReactUpdateQueue去更新,返回PublicInstance

如果不能就更新就unmount掉。

接着往下看，没有prevComponent和有而不能更新的话就执行ReactMount._renderNewRootComponent,
这个方法中最重要的就是调用 
```javascript
// The initial render is synchronous but any updates that happen during
// rendering, in componentWillMount or componentDidMount, will be batched
// according to the current batching strategy.

ReactUpdates.batchedUpdates(
  batchedMountComponentIntoNode,
  componentInstance,
  container,
  shouldReuseMarkup,
  context
);
```
