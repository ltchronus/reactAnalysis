### batchingStrategy
批量更新时总会调用ReactUpdates.batchedUpdates

```javascript
function batchedUpdates(callback, a, b, c, d, e) {
  ensureInjected();
  return batchingStrategy.batchedUpdates(callback, a, b, c, d, e);
}
```
这里有个batchingStrategy是什么👻？？

来看ensureInjected代码
```javascript
function ensureInjected() {
  invariant(
    ReactUpdates.ReactReconcileTransaction && batchingStrategy,
    'ReactUpdates: must inject a reconcile transaction class and batching ' +
    'strategy'
  );
}
```
说明这里必须要有batchingStrategy和ReactUpdates.ReactReconcileTransaction，那到底是什么时候inject的呢

ReactDOM 开始就inject了一些东西看看有没有
```javascript
ReactDOMInjection.inject();
ReactDOMStackInjection.inject();
```

ReactDOMInjection.inject里面inject了事件ReactEventListener，ReactDOMComponentTree等

ReactDOMStackInjection里就发现了
```javascript
ReactUpdates.injection.injectBatchingStrategy(
  ReactDefaultBatchingStrategy
);
```

说明batchedUpdates里的batchingStrategy就是ReactDefaultBatchingStrategy

我们看ReactDefaultBatchingStrategy源码
```javascript
/**
  * Call the provided function in a context within which calls to `setState`
  * and friends are batched such that components aren't updated unnecessarily.
  */
batchedUpdates: function(callback, a, b, c, d, e) {
  var alreadyBatchingUpdates = ReactDefaultBatchingStrategy.isBatchingUpdates;

  ReactDefaultBatchingStrategy.isBatchingUpdates = true;

  // The code is written this way to avoid extra allocations
  if (alreadyBatchingUpdates) {
    return callback(a, b, c, d, e);
  } else {
    return transaction.perform(callback, null, a, b, c, d, e);
  }
}
```
发现调用ReactUpdates.batchedUpdates就是为了调用transaction.perform