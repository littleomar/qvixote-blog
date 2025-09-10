Vue中的事件管理`scheduler`利用了浏览器的[[浏览器运行原理、宏任务&微任务|运行原理]]控制`effect`中的`callback`函数或者`nextTick`函数在合适的时机执行
#### `watch`函数的`flush`配置项
我们可以从`watch`函数的`flush`参数（`pre`、`post`和`sync`）入手，浅谈一下事件调度和管理，这三个参数也是vue事件管理的核心，分别对应着三个调度时机， 三者的执行顺序为`sync`、`pre`、`post`
```ts
// /core/packages/runtime-core/src/apiWatch.ts
function doWatch(
  source: WatchSource | WatchSource[] | WatchEffect | object,
  cb: WatchCallback | null,
  options: WatchOptions = EMPTY_OBJ,
): WatchHandle {
  // ...
  // scheduler
  let isPre = false
  if (flush === 'post') {  // 当flush === 'post'时会把job函数推到添加到postQueue中，当queue中的job清空后会才会清空此队列
    baseWatchOptions.scheduler = job => {
      queuePostRenderEffect(job, instance && instance.suspense)
    }
  } else if (flush !== 'sync') { // 默认为pre会把job添加到一个queue中，然后在微任务中清除这个队列
    // default: 'pre'
    isPre = true
    baseWatchOptions.scheduler = (job, isFirstRun) => {
      if (isFirstRun) {
        job()
      } else {
        queueJob(job)
      }
    }
  }
  // 当flush === 'sync'时候即不设置scheduler函数，当调用scheduler是如果为空则会直接执行job这就是flush === 'sync'为同步执行的原因，它的时机是在主线程中随JS代码一起执行
  // ...

  return watchHandle
}
```
#### `scheduler`模块的实现
在`scheduler.ts`文件中主要定义了一下几个变量和方法
```ts
// /core/packages/runtime-core/src/scheduler.ts
const queue: SchedulerJob[] = []  // 待更新队列

// 存储flush: post的队列
const pendingPostFlushCbs: SchedulerJob[] = []
let activePostFlushCbs: SchedulerJob[] | null = null

// 指向当前清空队列的微任务
let currentFlushPromise: Promise<void> | null = null 

// 清空队列函数，如果发现当前存在清空队列的微任务则不会进行否则创建微任务指向currentFlushPromise
// 每次向queue中添加新的任务时都会触发queueFlush不过有currentFlushPromise的检测，所以同时只会存在一个清空队列的Promise微任务
function queueFlush() {
  if (!currentFlushPromise) {
    currentFlushPromise = resolvedPromise.then(flushJobs)
  }
}
```
##### `pre`任务
从`watch`函数可以看出在`flush`类型为`pre`时，会调用`queueJob`将`job`推到`queue`队列中，并实现了排序操作
```ts
export function queueJob(job: SchedulerJob): void {
  if (!(job.flags! & SchedulerJobFlags.QUEUED)) {
    const jobId = getId(job)
    const lastJob = queue[queue.length - 1]
    if (
      !lastJob ||
      // fast path when the job id is larger than the tail
      (!(job.flags! & SchedulerJobFlags.PRE) && jobId >= getId(lastJob))
    ) {
      queue.push(job)
    } else {
      queue.splice(findInsertionIndex(jobId), 0, job)
    }

    job.flags! |= SchedulerJobFlags.QUEUED

    queueFlush()
  }
}
```
在`queueJob`中会按照`job.id`按照从小到大进行排序，这个`job.id`就等于组件实例的`uid`，因此`queue`队列会按照组件的`uid`排序，如果没有`job.id`则会根据是否有`SchedulerJobFlags.PRE`将任务排到首部或尾部
注意一下插入顺序的控制，这个`findInsertionIndex`比较有意思，首先它会按照组件的`id`大小进行排序，然后呢后来插入的拥有相同`id`的`job`会排在最后一位
```ts
// Use binary-search to find a suitable position in the queue. The queue needs
// to be sorted in increasing order of the job ids. This ensures that:
// 1. Components are updated from parent to child. As the parent is always
//    created before the child it will always have a smaller id.
// 2. If a component is unmounted during a parent component's update, its update
//    can be skipped.
// A pre watcher will have the same id as its component's update job. The
// watcher should be inserted immediately before the update job. This allows
// watchers to be skipped if the component is unmounted by the parent update.
function findInsertionIndex(id: number) {
  let start = flushIndex + 1
  let end = queue.length

  while (start < end) {
    const middle = (start + end) >>> 1
    const middleJob = queue[middle]
    const middleJobId = getId(middleJob)
    if (
      middleJobId < id ||
      (middleJobId === id && middleJob.flags! & SchedulerJobFlags.PRE)
    ) {
      start = middle + 1
    } else {
      end = middle
    }
  }

  return start
}
```
##### `post`任务
在`watch`函数相关代码中会看到`flush`为`post`时会调用`queuePostRenderEffect`不考虑`suspense`的情况下最终会调用`queuePostFlushCb`
```ts
// /core/packages/runtime-core/src/renderer.ts
export const queuePostRenderEffect: (
  fn: SchedulerJobs,
  suspense: SuspenseBoundary | null,
) => void = __FEATURE_SUSPENSE__
  ? __TEST__
    ? // vitest can't seem to handle eager circular dependency
      (fn: Function | Function[], suspense: SuspenseBoundary | null) =>
        queueEffectWithSuspense(fn, suspense)
    : queueEffectWithSuspense
  : queuePostFlushCb

// /core/packages/runtime-core/src/scheduler.ts
// 把post job推到pendingPostFlushCbs队列中
export function queuePostFlushCb(cb: SchedulerJobs): void {
  if (!isArray(cb)) {
    if (activePostFlushCbs && cb.id === -1) {  // 此处逻辑和rendererTemplateRef有关，先不予考虑
      activePostFlushCbs.splice(postFlushIndex + 1, 0, cb)
    } else if (!(cb.flags! & SchedulerJobFlags.QUEUED)) {  // 把job添加到pendingPostFlushCbs中并把job.flags标志置为SchedulerJobFlags.QUEUED，防重复添加
      pendingPostFlushCbs.push(cb)
      cb.flags! |= SchedulerJobFlags.QUEUED
    }
  } else {
    // if cb is an array, it is a component lifecycle hook which can only be
    // triggered by a job, which is already deduped in the main queue, so
    // we can skip duplicate check here to improve perf
    pendingPostFlushCbs.push(...cb)
  }
  queueFlush()
}
```
##### `queueFlush`清空`job`队列
```ts
// /core/packages/runtime-core/src/scheduler.ts
// queueFlush方法会开启一个微任务，在微任务中执行清空job的操作，currentFlushPromise指向当前微任务，如果存在则跳过
function queueFlush() {
  if (!currentFlushPromise) {
    currentFlushPromise = resolvedPromise.then(flushJobs)
  }
}

// 清空队列函数，包含pre队列和post队列
function flushJobs(seen?: CountMap) {
  if (__DEV__) { 
    seen = seen || new Map()
  }

  // conditional usage of checkRecursiveUpdate must be determined out of
  // try ... catch block since Rollup by default de-optimizes treeshaking
  // inside try-catch. This can leave all warning code unshaked. Although
  // they would get eventually shaken by a minifier like terser, some minifiers
  // would fail to do that (e.g. https://github.com/evanw/esbuild/issues/1610)
  const check = __DEV__
    ? (job: SchedulerJob) => checkRecursiveUpdates(seen!, job)
    : NOOP

  try {
    for (flushIndex = 0; flushIndex < queue.length; flushIndex++) {
      // 在对queue进行遍历的时候，调用job可能会产生新的job，此时会直接把job推到queue队列中，由于findInsertionIndex会为新推进来的job找到合适的位置，此时会放在同一id的尾部，之后会按顺序执行
      const job = queue[flushIndex]
      if (job && !(job.flags! & SchedulerJobFlags.DISPOSED)) {
        if (__DEV__ && check(job)) {
          continue
        }
        if (job.flags! & SchedulerJobFlags.ALLOW_RECURSE) {
          job.flags! &= ~SchedulerJobFlags.QUEUED
        }
        callWithErrorHandling(
          job,
          job.i,
          job.i ? ErrorCodes.COMPONENT_UPDATE : ErrorCodes.SCHEDULER,
        )
        if (!(job.flags! & SchedulerJobFlags.ALLOW_RECURSE)) {
          job.flags! &= ~SchedulerJobFlags.QUEUED
        }
      }
    }
  } finally {
    // If there was an error we still need to clear the QUEUED flags
    for (; flushIndex < queue.length; flushIndex++) {
      const job = queue[flushIndex]
      if (job) {
        job.flags! &= ~SchedulerJobFlags.QUEUED
      }
    }

    flushIndex = -1
    queue.length = 0

    // 清空post队列
    flushPostFlushCbs(seen)
    // 重置微任务函数
    currentFlushPromise = null
    // If new jobs have been added to either queue, keep flushing
    // 判断在执行flushPostFlushCbs之后会不会推进新的job
    if (queue.length || pendingPostFlushCbs.length) {
      flushJobs(seen)
    }
  }
}

// 清空post任务队列
export function flushPostFlushCbs(seen?: CountMap): void {
  if (pendingPostFlushCbs.length) {
    const deduped = [...new Set(pendingPostFlushCbs)].sort(
      (a, b) => getId(a) - getId(b),
    )
    pendingPostFlushCbs.length = 0

    // #1947 already has active queue, nested flushPostFlushCbs call
    if (activePostFlushCbs) {
      activePostFlushCbs.push(...deduped)
      return
    }
    // 去重后的postqueuejob赋值给activePostFlushCbs
    activePostFlushCbs = deduped
    if (__DEV__) {
      seen = seen || new Map()
    }

    // 与queue不同的是在调用job的过程中这个遍历是不会发生改变的，新产生的post job只会重新存储到pendingPostFlushCbs，直到下一轮flushJobs才会执行
    for (
      postFlushIndex = 0;
      postFlushIndex < activePostFlushCbs.length;
      postFlushIndex++
    ) {
      const cb = activePostFlushCbs[postFlushIndex]
      if (__DEV__ && checkRecursiveUpdates(seen!, cb)) {
        continue
      }
      if (cb.flags! & SchedulerJobFlags.ALLOW_RECURSE) {
        cb.flags! &= ~SchedulerJobFlags.QUEUED
      }
      if (!(cb.flags! & SchedulerJobFlags.DISPOSED)) cb()
      cb.flags! &= ~SchedulerJobFlags.QUEUED
    }
    activePostFlushCbs = null
    postFlushIndex = 0
  }
}
```
##### *伪* 宏任务&微任务
Vue3中的任务调度模块的设计思想和JS中的事件循环十分类似，抛开`flush === 'sync'`不说，因为同步执行也可以不算事件调度，那么我们就可以把`flush === 'pre'`也就是`effect`默认的模式，像是不加参数的`watch`和`render`函数诸如此类，可以比作JS中的微任务，而`flush === 'post'`看作是JS中的宏任务，不过这只是相对而言，并不是真正的宏任务&微任务，在清空`preJobs`队列过程中如果产生新的`preJob`则会插入到`queue`中按顺序执行，如果产生新的`postJob`则会等到`queue`队列清空完成后才开始执行，那么如果在清空`postJobs`的过程中产生新的`preJob`或者`postJob`会在此次`postJobs`（这个`postJobs`在遍历过程中是不会发生改变的）清空完成后，就会开启新一轮的`flushJobs`，注意`flushJobs`的伪事件循环都是在同一个微任务中运行的。参考宏任务&微任务再火焰图中的表现更容易理解[[浏览器运行原理、宏任务&微任务#JS执行|浏览器事件循环]]

下图的宏任务&微任务可以替换成`preJob`和`postJob`，原理和执行顺序是一样的
![image](https://origin.picgo.net/2025/09/08/imageaedfd40a90a8a825.png)
##### `nextTick`的原理
理解完浏览器的事件循环和vue中的`preJob&postJob`那么`nextTick`的执行时机就很简单了
```ts
// /core/packages/runtime-core/src/scheduler.ts
export function nextTick<T, R>(
  this: T,
  fn?: (this: T) => R | Promise<R>,
): Promise<void | R> {
  const p = currentFlushPromise || resolvedPromise
  return fn ? p.then(this ? fn.bind(this) : fn) : p
}
```
`nextTick`会在此次微任务执行完毕之后开启一个新的微任务，所以在执行`nextTick`之前会清空所有的`preJobs`和`postJobs，但是会在`setTimeout`宏任务之前执行

##### 主动清除队列
在组件初始化或者更新的过程中会主动调用`flushPreFlushCbs`和`flushPostFlushCbs`
```ts
export function flushPreFlushCbs(
  instance?: ComponentInternalInstance,
  seen?: CountMap,
  // skip the current job
  i: number = flushIndex + 1,
): void {
  if (__DEV__) {
    seen = seen || new Map()
  }
  for (; i < queue.length; i++) {
    const cb = queue[i]
    if (cb && cb.flags! & SchedulerJobFlags.PRE) {
      if (instance && cb.id !== instance.uid) {
        continue
      }
      if (__DEV__ && checkRecursiveUpdates(seen!, cb)) {
        continue
      }
      queue.splice(i, 1)
      i--
      if (cb.flags! & SchedulerJobFlags.ALLOW_RECURSE) {
        cb.flags! &= ~SchedulerJobFlags.QUEUED
      }
      cb()
      if (!(cb.flags! & SchedulerJobFlags.ALLOW_RECURSE)) {
        cb.flags! &= ~SchedulerJobFlags.QUEUED
      }
    }
  }
}

export function flushPostFlushCbs(seen?: CountMap): void {
  if (pendingPostFlushCbs.length) {
    const deduped = [...new Set(pendingPostFlushCbs)].sort(
      (a, b) => getId(a) - getId(b),
    )
    pendingPostFlushCbs.length = 0

    // #1947 already has active queue, nested flushPostFlushCbs call
    if (activePostFlushCbs) {
      activePostFlushCbs.push(...deduped)
      return
    }

    activePostFlushCbs = deduped
    if (__DEV__) {
      seen = seen || new Map()
    }

    for (
      postFlushIndex = 0;
      postFlushIndex < activePostFlushCbs.length;
      postFlushIndex++
    ) {
      const cb = activePostFlushCbs[postFlushIndex]
      if (__DEV__ && checkRecursiveUpdates(seen!, cb)) {
        continue
      }
      if (cb.flags! & SchedulerJobFlags.ALLOW_RECURSE) {
        cb.flags! &= ~SchedulerJobFlags.QUEUED
      }
      if (!(cb.flags! & SchedulerJobFlags.DISPOSED)) cb()
      cb.flags! &= ~SchedulerJobFlags.QUEUED
    }
    activePostFlushCbs = null
    postFlushIndex = 0
  }
}
```
如果调用`flushPreFlushCbs`时候传入`instance`则是指定组件，如果不传则是指的根组件，下面是调用时候的代码
```ts
// /core/packages/runtime-core/src/renderer.ts
// 指定组件
  const updateComponentPreRender = (
    instance: ComponentInternalInstance,
    nextVNode: VNode,
    optimized: boolean,
  ) => {
    nextVNode.component = instance
    const prevProps = instance.vnode.props
    instance.vnode = nextVNode
    instance.next = null
    updateProps(instance, nextVNode.props, prevProps, optimized)
    updateSlots(instance, nextVNode.children, optimized)

    pauseTracking()
    // props update may have triggered pre-flush watchers.
    // flush them before the render update.
    flushPreFlushCbs(instance)
    resetTracking()
  }
// 根组件
  const render: RootRenderFunction = (vnode, container, namespace) => {
    if (vnode == null) {
      if (container._vnode) {
        unmount(container._vnode, null, null, true)
      }
    } else {
      patch(
        container._vnode || null,
        vnode,
        container,
        null,
        null,
        null,
        namespace,
      )
    }
    container._vnode = vnode
    if (!isFlushing) {
      isFlushing = true
      flushPreFlushCbs()
      flushPostFlushCbs()
      isFlushing = false
    }
  }
```
具体应用场景？？？？？