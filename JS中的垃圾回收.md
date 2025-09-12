在javascript中，垃圾回收 Garbage Collection 简称GC，GC的过程是自动的，不需要人工干预，目的是回收内存，防止内存泄漏，与其他语言不同例如C，不需要手动分配或者回收内存。

> *引用计数（Reference Counting）属于比较老的方案，现代JS引擎中不再使用，所以下文不会在提*
#### 机制：（Mark-Sweep）标记清除
对根（全局变量、当前执行栈和闭包等）开始，遍历所有的可访问变量。未被访问的变量（对象）会被GC，回收内存

#### GC触发时机
- **内存不足**：当程序内存到达一定阈值时会触发GC，不过这个内存阈值是动态变化的，主要与当前环境和历史活动有关
- **空闲时**：尤其在进行网络请求等异步操作时，CPU占用量比较低，会进行GC避免对主线程的性能造成影响。

#### 现代执行环境中GC的优化机制
##### 分代回收 Generational GC
在**V8**引擎中实现，原理：[代际假说(Generational Hypothesis)](https://v8.dev/blog/trash-talk)。变量划分为两个部分，新生代和旧生代，新生代又把内存划分为两个半区，在进行垃圾回收时会将存活的变量复制到另一个半区，未被引用的变量直接丢弃，经过两次复制依然存活的变量则会提升到旧生代中，旧生代中的变量则会采用Mark-Sweep的机制进行GC。
![image](https://origin.picgo.net/2025/09/01/image54230cfaa46d89b9.png)
##### 三色标记

把对象分为白、灰、黑三种，在数据初始化的时候所有对象都是白色的，然后从根（全局变量、运行栈、闭包）中开始遍历，把可以直接访问的对象标记为灰色并把它放到一个待处理的队列中，下一步遍历上述队列中的灰色数据，如果子对象都可达则标记为黑色，如果子对象中有白色对象，则保持灰色不变，重复上述过程，直至没有灰色对象。标记阶段结束后开启回收机制，黑色对象为依然存活的对象，白色对象为无用对象则会被清理。借用wiki中的一张图：
![Animation_of_tri-color_garbage_collection](https://origin.picgo.net/2025/09/01/Animation_of_tri-color_garbage_collection5730391962d204a9.gif)
##### 其他
- 增量标记：分段进行标记，GC 标记过程被拆分成多个小片段（约 5–10 ms），减少主线程的长时间占用。
- 并行与并发：主线程 + 多线程 + 后台线程进行标记
#### 错误使用&建议
虽然在JS中垃圾回收机制是自动的，但是错误或不规范使用仍会导致内存泄漏：
- 不使用 `var let const` 声明变量导致变量自动变为全局变量，此变量会持续存在内存中，无法正常回收。
- `setInterval` 和 `setTimeout` 在使用完成后需手动清除，否则会持续引用无法释放内存
- DOM中如果有添加的监听器 `addEventListener` ,当相关DOM被移除时，所绑定的监听事件是不会自动移除的，需要手动移除事件监听器`removeEventListener` 
- 使用`document.body.removeChild(div)`移除DOM时DOM不一定会被回收，例如此DOM在被变量引用时
- 闭包持续引用外部作用域中的大型对象，即使该对象已不再使用，此对象也不会被回收
```javascript
function closureLeaks() {
  const big = new Array(100000).fill('data');
  return function() { console.log(big[0]); };
}
const fn = closureLeaks(); 
fn()  // `big` 不会被回收
```
- 在使用 `Map` 或者 `Set` 时可能会妨碍垃圾回收，在合适的情况下建议使用 `WeakMap` 或 `WeakSet`
#### Vue 中和GC相关的内容
[[Vue响应式的版本控制#依赖清理|依赖清理]]