在Vue响应式中和版本相关的定义有两个`Link.version`和`Dep.version`
#### `version`的初始化
两者的初始化，`Dep`的初始`version`是`0`，`Linik`的`version`则是同步`Dep.version`
```ts
// /core/packages/reactivity/src/dep.ts
// Link中version的定义和初始化
export class Link {
  /**
   * - Before each effect run, all previous dep links' version are reset to -1
   * - During the run, a link's version is synced with the source dep on access
   * - After the run, links with version -1 (that were never used) are cleaned
   *   up
   */
  version: number

  constructor(
    public sub: Subscriber,
    public dep: Dep,
  ) {
    this.version = dep.version
    // ...
  }
}
// Dep中version的定义和初始化
export class Dep {
  version = 0
  // ...

  constructor(public computed?: ComputedRefImpl | undefined) {
    if (__DEV__) {
      this.subsHead = undefined
    }
  }
}
```
#### `Dep.version`的修改&用途
字面意思*版本*，当响应式数据发生改变时他所代表的`Dep`版本也会同步发生自增修改，在`effect`函数更新时会调用相应是数据的`get`方法，此时会把`link`的版本与`dep`版本同步，在调用`get`之前会把所有的`link`版本设置为`-1`，如果`effect`未调用版本同步逻辑则证明此`link`未被引用，因此需要**依赖清理**。如果发生同步，则此处`link`所记录的版本就是最终版本，在响应式数据发生修改时`Dep.version`自增，`link.version`则可以看作是旧版本，因此可以根据`dep.version`与`link.version`的比较来判断`effect`即（`render`，`watch`，`computed`）是否真正需要更新
##### `isDirty` 对比前后版本判断是否需要更新
关于响应式数据版本的变化，vue中定义了一个`isDirty`的方法来对比前后的版本，并据此判断是否需要进行真正的更新操作
```ts
// isDirty
// /core/packages/reactivity/src/effect.ts
// isDirty函数的定义，如果对比出版本差异则标记为dirty，如果版本相同则标记为非dirty
function isDirty(sub: Subscriber): boolean {
  for (let link = sub.deps; link; link = link.nextDep) {
    if (
      link.dep.version !== link.version ||  // 比较前后版本如果是普通ref或者reactive类型的值发生修改则此处会不想等，直接返回true
      (link.dep.computed &&
        (refreshComputed(link.dep.computed) ||
          link.dep.version !== link.version))  // 如果是computed则会通过refreshComputed获取computed最新的值，如果发生改变则会更新dep.version然而此时dep.version的值还未同步给link.version因此会产生新旧版本的差异，标记为dirty
    ) {
      return true
    }
  }
  // @ts-expect-error only for backwards compatibility where libs manually set
  // this flag - e.g. Pinia's testing module
  if (sub._dirty) {
    return true
  }
  return false
}
```
以下是`computed`和`ref`、`reactive`中的版本同步操作
```ts
// /core/packages/reactivity/src/computed.ts
// computed 实例中get的定义，完成value值的读取之后同步link和dep的版本
export class ComputedRefImpl<T = any> implements Subscriber {
  //...
  get value(): T {
    const link = __DEV__
      ? this.dep.track({
          target: this,
          type: TrackOpTypes.GET,
          key: 'value',
        })
      : this.dep.track()
    refreshComputed(this)
    // sync version after evaluation
    if (link) {
      link.version = this.dep.version
    }
    return this._value
  }
  // ...
}

// /core/packages/reactivity/src/dep.ts
// 在响应式的get方法或者proxy.get拦截器中会调用dep.track,其中包含version同步的逻辑
export class Dep {
  // ... 
  track(debugInfo?: DebuggerEventExtraInfo): Link | undefined {
    // ...

    let link = this.activeLink
    if (link === undefined || link.sub !== activeSub) {
      
    } else if (link.version === -1) {
      // reused from last run - already a sub, just sync version
      link.version = this.version
      // ...
    }

    return link
  }
  // ...
}
```
##### 依赖清理
在进行依赖收集之前会调用`prepareDeps`方法为依赖收集作准备，在[[learn vue/Vue响应式的依赖收集|依赖收集]]文中有分析过，过程中会调用相应是数据的`get`拦截器，而`get`的过程中会同步`version`版本，假如在调用`get`之前也就是`prepareDeps`过程中将`link`版本重置为`-1`，而`get`过程却没有同步`version`的情况下，则`version`依然保持为`-1`，则为`-1`的`version`就成了被清理的对象
```ts
function prepareDeps(sub: Subscriber) {
  // Prepare deps for tracking, starting from the head
  for (let link = sub.deps; link; link = link.nextDep) {
    // set all previous deps' (if any) version to -1 so that we can track
    // which ones are unused after the run
    link.version = -1
    // store previous active sub if link was being used in another context
    link.prevActiveLink = link.dep.activeLink
    link.dep.activeLink = link
  }
}

// /core/packages/reactivity/src/effect.ts
// 解除引用的相关操作，解除引用之后系统会自动调用GC进行垃圾回收释放内存
function cleanupDeps(sub: Subscriber) {
  // Cleanup unsued deps
  let head
  let tail = sub.depsTail
  let link = tail
  while (link) {
    const prev = link.prevDep
    if (link.version === -1) {
      if (link === tail) tail = prev
      // unused - remove it from the dep's subscribing effect list
      removeSub(link)
      // also remove it from this effect's dep list
      removeDep(link)
    } else {
      // The new head is the last node seen which wasn't removed
      // from the doubly-linked list
      head = link
    }

    // restore previous active link if any
    link.dep.activeLink = link.prevActiveLink
    link.prevActiveLink = undefined
    link = prev
  }
  // set the new head & tail
  sub.deps = head
  sub.depsTail = tail
}
```
JS中的[[JS中的垃圾回收|垃圾回收]]属于浏览器的自动行为，因此我们只需要解除引用即可，在结束`version`标记之后会遍历整个`deps`链表，如果`link.version === -1`则需要清除这个无用节点，清除`LinkNode`需要解除其在`subs`链表和`deps`链表中的引用分别对应`removeSub(link)`和`removeDep(link)`，清除依赖的过程就体现了[[Vue中的双向链表|双向链表]]的优势，能够快速定位，减少遍历所带来的开销。
```ts
// /core/packages/reactivity/src/effect.ts
// 移除subs链上关于此LinkNode的引用
function removeSub(link: Link, soft = false) {
  const { dep, prevSub, nextSub } = link
  if (prevSub) {
    prevSub.nextSub = nextSub
    link.prevSub = undefined
  }
  if (nextSub) {
    nextSub.prevSub = prevSub
    link.nextSub = undefined
  }
  if (__DEV__ && dep.subsHead === link) {
    // was previous head, point new head to next
    dep.subsHead = nextSub
  }

  if (dep.subs === link) {
    // was previous tail, point new tail to prev
    dep.subs = prevSub

    if (!prevSub && dep.computed) {
      // if computed, unsubscribe it from all its deps so this computed and its
      // value can be GCed
      dep.computed.flags &= ~EffectFlags.TRACKING
      for (let l = dep.computed.deps; l; l = l.nextDep) {
        // here we are only "soft" unsubscribing because the computed still keeps
        // referencing the deps and the dep should not decrease its sub count
        removeSub(l, true)
      }
    }
  }

  if (!soft && !--dep.sc && dep.map) {
    // #11979
    // property dep no longer has effect subscribers, delete it
    // this mostly is for the case where an object is kept in memory but only a
    // subset of its properties is tracked at one time
    dep.map.delete(dep.key)
  }
}

// 移除deps链上关于此LinkNode的引用
function removeDep(link: Link) {
  const { prevDep, nextDep } = link
  if (prevDep) {
    prevDep.nextDep = nextDep
    link.prevDep = undefined
  }
  if (nextDep) {
    nextDep.prevDep = prevDep
    link.nextDep = undefined
  }
}
```
#### `globalVersion`
`globalVersion`存在的意义就是能够快速判断`computed`是否发生了修改
```ts
export function refreshComputed(computed: ComputedRefImpl): undefined {
  // ...
  // Global version fast path when no reactive changes has happened since
  // last refresh.
  if (computed.globalVersion === globalVersion) {
    return
  }
  computed.globalVersion = globalVersion
  // ...
}
```
在`Dep.trigger()`即`dep`所对应的`value`发生修改时`globalVersion`也会发生自增（这些触发`globalVersion++`的`dep`或许和`computed`无关），在`refreshComputed`时候会对比`computed`的`version`是否和`globalVersion`是否保持一致，如果一致则说明`computed`不需要执行`effect.callback`反之则同步版本号，进而进行进一步比较。



