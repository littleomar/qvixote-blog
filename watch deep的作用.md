💡思考：为什么需要 `deep` 对象不是已经被 `Proxy` 代理了吗？为什么修改对象属性的时候不能正确监听？

这个问题的关键在于被 `watch` 的对象内的属性未被正确收集

Example：
```ts
const targetObj = ref({
	one: 1,
	two: 2,
	three: 3
})

watch(() => targetObj.value, _ => {
	// do something
})
```

可以看出上述的 `getter` 函数，也就是`() => targetObj.value` 只会触发 `targetObj Ref` 的 `get` 拦截器，因此，在进行依赖收集时 `effect` 只会将 `targetObj.value` 作为依赖，然而在内部属性发生改变时是不会触发`watch` 的。

添加 `deep` 属性后，会在调用 `getter` 函数时遍历`traverse`读取 `targetObj` 中的所有属性，属性被调用所以会触发`Proxy`中的`get`拦截器进行依赖收集，达到监听的目的。