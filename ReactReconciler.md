### ReactReconciler

ReactReconciler到底是个什么👻呢？

官网介绍
> ReactReconciler is a wrapper with mountComponent, receiveComponent(), and unmountComponent() methods. 
> It calls the underlying implementations on the internal instances, but also includes some code around them 
> that is shared by all internal instance implementations.

ReactReconciler就是做mountComponent, receiveComponent, unmountComponent


伪代码类结构如下
```javascript
class ReactReconciler {
  mountComponent() {}

  getHostNode() {}

  unmountComponent() {}

  receiveComponent() {}

  performUpdateIfNecessary() {}
}
```

其中mountComponent初始化了internal component instances，并将这些事例render成markup和注册事件。
```javascript
/**
   * Initializes the component, renders markup, and registers event listeners.
   *
   * @param {ReactComponent} internalInstance
   * @param {ReactReconcileTransaction|ReactServerRenderingTransaction} transaction
   * @param {?object} the containing host component instance
   * @param {?object} info about the host container
   * @return {?string} Rendered markup to be inserted into the DOM.
   * @final
   * @internal
   */
  mountComponent: function(
    internalInstance,
    transaction,
    hostParent,
    hostContainerInfo,
    context,
    parentDebugID // 0 in production and for roots
  ) {

    var markup = internalInstance.mountComponent(
      transaction,
      hostParent,
      hostContainerInfo,
      context,
      parentDebugID
    );
    if (internalInstance._currentElement &&
        internalInstance._currentElement.ref != null) {
      transaction.getReactMountReady().enqueue(attachRefs, internalInstance);
    }

    return markup;
  },
```

这里很直接，调用internalInstance.mountComponent