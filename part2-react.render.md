### Part 2 React.render
接第一部分，执行到ReactUpdates.batchedUpdates，如下
```javascript
ReactUpdates.batchedUpdates(
  batchedMountComponentIntoNode,
  componentInstance,
  container,
  shouldReuseMarkup,
  context
);
```
看过batchedUpdates中batchedUpdates和Transaction我们知道，这里只是batchedMountComponentIntoNode
包装执行而已，直接看batchedMountComponentIntoNode

```javascript
/**
 * Batched mount.
 *
 * @param {ReactComponent} componentInstance The instance to mount.
 * @param {DOMElement} container DOM element to mount into.
 * @param {boolean} shouldReuseMarkup If true, do not insert markup
 */
function batchedMountComponentIntoNode(
  componentInstance,
  container,
  shouldReuseMarkup,
  context
) {
  var transaction = ReactUpdates.ReactReconcileTransaction.getPooled(
    /* useCreateElement */
    !shouldReuseMarkup && ReactDOMFeatureFlags.useCreateElement
  );
  transaction.perform(
    mountComponentIntoNode,
    null,
    componentInstance,
    container,
    transaction,
    shouldReuseMarkup,
    context
  );
  ReactUpdates.ReactReconcileTransaction.release(transaction);
}
```

这里我们又看到一个ReactReconcileTransaction子类，从名字上可以看出来它是在ReactReconciler过程中用到的
后面肯定要调用ReactReconciler，看看它到底wapper了啥

ReactReconcileTransaction源码中有3个wapper
```javascript
var TRANSACTION_WRAPPERS = [
  SELECTION_RESTORATION,
  EVENT_SUPPRESSION,
  ON_DOM_READY_QUEUEING,
];
```
做了一下3件事
```javascript
 * - The order that these are listed in the transaction is critical:
 * - Suppresses events.
 * - Restores selection range.
 ```
* SELECTION_RESTORATION
> Ensures that, when possible, the selection range (currently selected text
> input) is not disturbed by performing the transaction.
字面意思预防input中文字选择影响到transaction.perform

* EVENT_SUPPRESSION
> Suppresses events (blur/focus) that could be inadvertently dispatched due to
> high level DOM manipulations (like temporarily removing a text input from the
> DOM).
预防DOM操作影响到events

* ON_DOM_READY_QUEUEING
> Provides a queue for collecting `componentDidMount` and
> `componentDidUpdate` callbacks during the transaction.
操作reactMountReady这个CallbackQueue 为生命周期中componentDidMount和componentDidUpdate是否可以执行
提供判断依据

并且它有3个比较重要的属性renderToStaticMarkup， reactMountReady，useCreateElement在后面ReactReconciler

这个ReactReconcileTransaction.perform又调用mountComponentIntoNode
```javascript
/**
 * Mounts this component and inserts it into the DOM.
 *
 * @param {ReactComponent} componentInstance The instance to mount.
 * @param {DOMElement} container DOM element to mount into.
 * @param {ReactReconcileTransaction} transaction
 * @param {boolean} shouldReuseMarkup If true, do not insert markup
 */
function mountComponentIntoNode(
  wrapperInstance,
  container,
  transaction,
  shouldReuseMarkup,
  context
) {
  var markerName;
  if (ReactFeatureFlags.logTopLevelRenders) {
    var wrappedElement = wrapperInstance._currentElement.props.child;
    var type = wrappedElement.type;
    markerName = 'React mount: ' + (
      typeof type === 'string' ? type :
      type.displayName || type.name
    );
    console.time(markerName);
  }

  var markup = ReactReconciler.mountComponent(
    wrapperInstance,
    transaction,
    null,
    ReactDOMContainerInfo(wrapperInstance, container),
    context,
    0 /* parentDebugID */
  );

  if (markerName) {
    console.timeEnd(markerName);
  }

  wrapperInstance._renderedComponent._topLevelWrapper = wrapperInstance;
  ReactMount._mountImageIntoNode(
    markup,
    container,
    wrapperInstance,
    shouldReuseMarkup,
    transaction
  );
}
```

这个方法最重要的就是
* ReactReconciler.mountComponent
  生成markup
* ReactMount._mountImageIntoNode
  将markup插入container中

到这里终于到达了ReactReconciler的调用了