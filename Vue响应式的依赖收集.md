在Vue初始化的过程中也会按照顺序初始化组件，执行组件内的`setup`函数，`setup`运行的时候就会调用`computed`、`watch`和`render`，记录相关的`fn`函数，在合适的时机调用，且在这些`fn`的调用过程中会读取`ref`或者`reactive`的值触发拦截器，进而触发依赖收集
**依赖收集**操作定义在`dep.ts`文件的`track`函数中，其中有两个`track`一个挂载到`Dep`实例上的`track`是给`ref`类型的响应式数据使用的，而另一个导出的`track`是给`reactive`类型的响应式数据使用的
#### `Dep`的定义
Vue中的每一个响应式数据都会关联一个这样的`Dep`实例，每个`ref`包括`computed`都对应一个`dep`，而`reactive`中的每一个键值都会对应一个`dep`不过创建新的`dep`实例是惰性的，只有当被访问时才会创建
```typescript
// /core/packages/reactivity/src/dep.ts
export class Dep {
  version = 0
  /**
   * Link between this dep and the current active effect
   */
  activeLink?: Link = undefined

  // 每个Dep实例都会保存subs链表，用于存储每个Dep的所有订阅者
  // 并且subs一直指向链表尾部
  /**
   * Doubly linked list representing the subscribing effects (tail)
   */
  subs?: Link = undefined

  /**
   * Doubly linked list representing the subscribing effects (head)
   * DEV only, for invoking onTrigger hooks in correct order
   */
  subsHead?: Link

  /**
   * For object property deps cleanup
   */
  map?: KeyToDepMap = undefined
  key?: unknown = undefined

  /**
   * Subscriber counter
   */
  sc: number = 0

  /**
   * @internal
   */
  readonly __v_skip = true
  // TODO isolatedDeclarations ReactiveFlags.SKIP

  constructor(public computed?: ComputedRefImpl | undefined) {
  }

  track(debugInfo?: DebuggerEventExtraInfo): Link | undefined {
    // 依赖收集的实现
  }
  // ...
}
```
#### `Dep.track()`
`Dep`实例中挂载的`track`用于构造`subs`链，控制`Dep.version`，链表排序等，当执行`effect.fn()`时候会有一个全局指针`activeSub`指向这个`subscriber`，可以叫做激活的`sub`
```typescript
// /core/packages/reactivity/src/dep.ts
export class Dep {
  // ...
  track(debugInfo?: DebuggerEventExtraInfo): Link | undefined {
    if (!activeSub || !shouldTrack || activeSub === this.computed) {
      return
    }

    let link = this.activeLink
    if (link === undefined || link.sub !== activeSub) {
      // 在收集依赖时候会对比此时Link节点中的dep和sub如果这两者不同则会创建新的Link
      link = this.activeLink = new Link(activeSub, this)
      // 更新effect中的deps链路并且将
      // add the link to the activeEffect as a dep (as tail)
      if (!activeSub.deps) {
        activeSub.deps = activeSub.depsTail = link
      } else {
        link.prevDep = activeSub.depsTail
        activeSub.depsTail!.nextDep = link
        activeSub.depsTail = link
      }

      addSub(link)
    } else if (link.version === -1) {
      // 更新逻辑
    }

    return link
  }
  // ...
}

function addSub(link: Link) {
  link.dep.sc++  // sc为计数器，每次被引用时都会+1
  if (link.sub.flags & EffectFlags.TRACKING) {
    const computed = link.dep.computed
    // computed getting its first subscriber
    // enable tracking + lazily subscribe to all its deps
    if (computed && !link.dep.subs) {
      computed.flags |= EffectFlags.TRACKING | EffectFlags.DIRTY
      for (let l = computed.deps; l; l = l.nextDep) {
        addSub(l)
      }
    }

    const currentTail = link.dep.subs
    if (currentTail !== link) {
      link.prevSub = currentTail
      if (currentTail) currentTail.nextSub = link
    }

    if (__DEV__ && link.dep.subsHead === undefined) {
      link.dep.subsHead = link
    }

    link.dep.subs = link
  }
}
```

在源码中可以看出`dep.subs`和`sub.deps`这两个链表都是在依赖收集时进行添加的
下图为在一个`effect`中引用三个不同`ref`的情形
![effect中的deps](https://origin.picgo.net/2025/09/06/effectdeps29ec3dedaba3af88.gif)
下图为在一个多个`effect`中引用同一个`ref`的情形，包含`activeSub`的指向
![Dep中的subs链表](https://origin.picgo.net/2025/09/07/Depsubs06fc23fd1bed173e.gif)
`computed`也属于`ref`并且会会维护一个`dep`实例，因此也会使用`Dep.track()` -> `addSub`收集依赖

#### 响应式`Object`调用的`track`
响应式`Object`也就是`reactive`在进行**依赖收集**时会调用这个前置`track`然后根据创建或者获取`Dep`再调用最终的`Dep.track()`
首先会创建一个全局的`targetMap`并把类型定义为`WeakMap`，使用`WeakMap`有利于GC，然后会将`target => Map`作为健值对存储到`targetMap`，而这个`Map`则是存储 `key => Dep`，所以每个响应式`Object`的`key`都对应一个`Dep`实例，在这个dep实例中会存储`map`和`key`的信息，方便之后的**依赖清理**

```typescript
export const targetMap: WeakMap<object, KeyToDepMap> = new WeakMap()
/**
 * Tracks access to a reactive property.
 *
 * This will check which effect is running at the moment and record it as dep
 * which records all effects that depend on the reactive property.
 *
 * @param target - Object holding the reactive property.
 * @param type - Defines the type of access to the reactive property.
 * @param key - Identifier of the reactive property to track.
 */
export function track(target: object, type: TrackOpTypes, key: unknown): void {
  if (shouldTrack && activeSub) {
    let depsMap = targetMap.get(target)
    if (!depsMap) {
      targetMap.set(target, (depsMap = new Map()))
    }
    let dep = depsMap.get(key)
    if (!dep) {
      depsMap.set(key, (dep = new Dep()))
      dep.map = depsMap
      dep.key = key
    }
    if (__DEV__) {
      dep.track({
        target,
        type,
        key,
      })
    } else {
      dep.track()
    }
  }
}
```