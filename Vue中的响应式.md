Vue的响应式模块（reactivity）是由**订阅者**（ subscriber）和**依赖**（dependence）组成的，vue3.5之后这两者通过链表节点相关联，对于订阅者来说其中有个属性为`deps： Link`依赖链，而对于依赖来说其中有一个属性为`subs: Link`订阅链，具体参照[[Vue中的双向链表]]，**依赖**`Dep`内主要定义**依赖收集**和**派发更新**的函数方法，**订阅者**则是负责触发**依赖收集**并接收**派发更新**的通知，然后调用实际更新函数
#### 响应式数据类型
在vue中所有的响应式数据都会包含`dep`类，所以只要在全局搜索`new Dep`就能找到都有哪种响应式数据了，在源码中可以看到响应式数据分为两类`Ref`类和`Reactive`类这两个类型实现原理有所不同
```typescript
// /core/packages/reactivity/src/ref.ts 
// 1. Ref 的定义
// 会在每个ref实例都会对应一个dep实例
class RefImpl<T = any> {
  _value: T
  private _rawValue: T

  dep: Dep = new Dep()

  public readonly [ReactiveFlags.IS_REF] = true
  public readonly [ReactiveFlags.IS_SHALLOW]: boolean = false

  constructor(value: T, isShallow: boolean) {
    this._rawValue = isShallow ? value : toRaw(value)
    this._value = isShallow ? value : toReactive(value)
    this[ReactiveFlags.IS_SHALLOW] = isShallow
  }

  get value() {
//    ...
  }

  set value(newValue) {
//    ...
  }
}

// /core/packages/reactivity/src/reactive.ts
// 2. reactive中的定义
// 此函数会在reactive类型中的值被访问时触发，对每个被访问的键值都会创建一个Dep实例
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

// /core/packages/reactivity/src/computed.ts
// 3. ComputedRefImpl 的定义 
// 在使用computed函数时会创建一个‘ComputedRefImpl’实例，他在响应式方面的实现原理和Ref类似，因此也可以看作是一类
export class ComputedRefImpl<T = any> implements Subscriber {
  /**
   * @internal
   */
  _value: any = undefined
  /**
   * @internal
   */
  readonly dep: Dep = new Dep(this)
  // ...
}
```
##### Ref类
在源码里可以看出`computed`和`ref`都属于**Ref类**，在Ref类的构造函数中会将ref的value值通过调用`toReactive`将value转为响应式，如果value类型为原始类型（Primitive types）数据（string, number, bigint, boolean, undefined, symbol, null）即非`Object`类型直接返回，`Object`则调用`reactive`转为响应式。那么原始类型数据具备**响应式**的条件是通过**ES6**中对类的`get`和`set`定义来实现的
```typescript
/**
 * Returns a reactive proxy of the given value (if possible).
 *
 * If the given value is not an object, the original value itself is returned.
 *
 * @param value - The value for which a reactive proxy shall be created.
 */
export const toReactive = <T extends unknown>(value: T): T =>
  isObject(value) ? reactive(value) : value
```
![image](https://origin.picgo.net/2025/09/06/imagea27a19c96d928ece.png)
##### Reactive类
**Reactive类**是将`Object`类型是数据变为响应式，其实现原理得益于**ES6**对对象拦截器`Proxy`的支持，`Proxy`会在`Object`之前加上一层代理。通过`Proxy`代理的对象是惰性的，只有在对象那的键值被访问时才会起作用
```typescript
// /core/packages/reactivity/src/reactive.ts
export function reactive(target: object) {
  // if trying to observe a readonly proxy, return the readonly version.
  if (isReadonly(target)) {
    return target
  }
  return createReactiveObject(
    target,
    false,
    mutableHandlers,
    mutableCollectionHandlers,
    reactiveMap,
  )
}
// /core/packages/reactivity/src/baseHandlers.ts
export const mutableHandlers: ProxyHandler<object> =
  /*@__PURE__*/ new MutableReactiveHandler()
  
// /core/packages/reactivity/src/baseHandlers.ts
// 定义拦截器中的'set' 当响应式数据被更新时触发'派发更新'操作
class MutableReactiveHandler extends BaseReactiveHandler {
  constructor(isShallow = false) {
    super(false, isShallow)
  }

  set(
    target: Record<string | symbol, unknown>,
    key: string | symbol,
    value: unknown,
    receiver: object,
  ): boolean {
  
    // ...
    const result = Reflect.set(
      target,
      key,
      value,
      isRef(target) ? target : receiver,
    )
    // 触发更新
    // don't trigger if target is something up in the prototype chain of original
    if (target === toRaw(receiver)) {
      if (!hadKey) {
        trigger(target, TriggerOpTypes.ADD, key, value)
      } else if (hasChanged(value, oldValue)) {
        trigger(target, TriggerOpTypes.SET, key, value, oldValue)
      }
    }
    return result
  }
  
  // ...
}

// /core/packages/reactivity/src/baseHandlers.ts
class BaseReactiveHandler implements ProxyHandler<Target> {
  constructor(
    protected readonly _isReadonly = false,
    protected readonly _isShallow = false,
  ) {}

  get(target: Target, key: string | symbol, receiver: object): any {
    if (key === ReactiveFlags.SKIP) return target[ReactiveFlags.SKIP]

    const isReadonly = this._isReadonly,
      isShallow = this._isShallow
    if (key === ReactiveFlags.IS_REACTIVE) {
      return !isReadonly
    } else if (key === ReactiveFlags.IS_READONLY) {
      return isReadonly
    } else if (key === ReactiveFlags.IS_SHALLOW) {
      return isShallow
    } else if (key === ReactiveFlags.RAW) {
      if (
        receiver ===
          (isReadonly
            ? isShallow
              ? shallowReadonlyMap
              : readonlyMap
            : isShallow
              ? shallowReactiveMap
              : reactiveMap
          ).get(target) ||
        // receiver is not the reactive proxy, but has the same prototype
        // this means the receiver is a user proxy of the reactive proxy
        Object.getPrototypeOf(target) === Object.getPrototypeOf(receiver)
      ) {
        return target
      }
      // early return undefined
      return
    }

    const targetIsArray = isArray(target)

    if (!isReadonly) {
      let fn: Function | undefined
      if (targetIsArray && (fn = arrayInstrumentations[key])) {
        return fn
      }
      if (key === 'hasOwnProperty') {
        return hasOwnProperty
      }
    }

    const res = Reflect.get(
      target,
      key,
      // if this is a proxy wrapping a ref, return methods using the raw ref
      // as receiver so that we don't have to call `toRaw` on the ref in all
      // its class methods
      isRef(target) ? target : receiver,
    )

    if (isSymbol(key) ? builtInSymbols.has(key) : isNonTrackableKeys(key)) {
      return res
    }

    if (!isReadonly) {
      track(target, TrackOpTypes.GET, key)
    }

    if (isShallow) {
      return res
    }

    if (isRef(res)) {
      // ref unwrapping - skip unwrap for Array + integer key.
      return targetIsArray && isIntegerKey(key) ? res : res.value
    }

    if (isObject(res)) {
      // Convert returned value into a proxy as well. we do the isObject check
      // here to avoid invalid value warning. Also need to lazy access readonly
      // and reactive here to avoid circular dependency.
      return isReadonly ? readonly(res) : reactive(res)
    }

    return res
  }
}
```
值得注意的是对数组操作的一些方法也会被'get'拦截器所拦截，数组方法拦截器做的具体事情不仅包括对默认方法的实现，假如此方法会改变数组的值，则在拦截器中会标记为不需要进行依赖收集，因为在数组下标和长度发生改变时候会统一处理依赖收集相关逻辑，这样可以避免依赖的重复收集
```typescript
// /core/packages/reactivity/src/arrayInstrumentations.ts
// 不需要依赖收集
export const arrayInstrumentations: Record<string | symbol, Function> = <any>{

  // ...  noTracking
  pop() {
    return noTracking(this, 'pop')
  },

  push(...args: unknown[]) {
    return noTracking(this, 'push', args)
  }
  // ...
}
```
![image](https://origin.picgo.net/2025/09/06/imageec26e1e94152c91c.png)
#### 响应式运行机制
上一节讲了`Dep`的定义和`Dep`主要负责的内容，**响应式运行机制**则是在合适的时机调用`Dep`内定义的方法，在vue中是通过effect完成的也就是监听者subscriber，vue中的effect所能调用的函数分为三种，`render`函数，`watch`函数和`computed`，是的这里也有`computed`，所以`computed`即是`ref`也是`effect`，每个`effect`都有2个核心的函数`fn`和`callback`，`fn`的主要作用是调用时进行**依赖收集**，`callback`则主要是**派发更新**时最终调用的执行函数
```typescript
// /core/packages/runtime-core/src/renderer.ts
// 创建effect关联组件实例，在render类型的effect中 render函数既当作fn又当作callback
// 此处的render函数就是平时使用tsx时setup所返回的函数
// 此函数的执行结果会返回VNode类型，这也是下边所说的VNodeTree
// 1. render类型的effect定义
  const setupRenderEffect: SetupRenderEffectFn = (
    instance,
    initialVNode,
    container,
    anchor,
    parentSuspense,
    namespace: ElementNamespace,
    optimized,
  ) => {
    const componentUpdateFn = () => {
      if (!instance.isMounted) {
      // ...
        if (el && hydrateNode) {
          // ...
        } else {
          // ...
          // 调用组件的render函数获取组成组件的VNodeTree
          const subTree = (instance.subTree = renderComponentRoot(instance))
          // 把Vnode转换为真实dom并挂载到dom树的过程
          patch(
            null,
            subTree,
            container,
            anchor,
            instance,
            parentSuspense,
            namespace,
          )
          // ...
        }
      } else {
        // render() & patch()
      }
    }

    // create reactive effect for rendering
    // 创建effect，并把effect挂载到组件实例上
    // ...
    const effect = (instance.effect = new ReactiveEffect(componentUpdateFn))
    // ...
  }
  
// /core/packages/reactivity/src/watch.ts
// 2. watch函数的定义
export function watch(
  source: WatchSource | WatchSource[] | WatchEffect | object,
  cb?: WatchCallback | null,
  options: WatchOptions = EMPTY_OBJ,
): WatchHandle {
  const { immediate, deep, once, scheduler, augmentJob, call } = options
  
  // 格式化getter函数 此处的getter即为上边说的fn
  if (isRef(source)) {
    getter = () => source.value
    forceTrigger = isShallow(source)
  } else if (isReactive(source)) {
    getter = () => reactiveGetter(source)
    forceTrigger = true
  } else if (isArray(source)) {
    isMultiSource = true
    forceTrigger = source.some(s => isReactive(s) || isShallow(s))
    getter = () =>
      source.map(s => {
        if (isRef(s)) {
          return s.value
        } else if (isReactive(s)) {
          return reactiveGetter(s)
        } else if (isFunction(s)) {
          return call ? call(s, WatchErrorCodes.WATCH_GETTER) : s()
        } else {
          __DEV__ && warnInvalidSource(s)
        }
      })
  } else if (isFunction(source)) {
    if (cb) {
      // getter with cb
      getter = call
        ? () => call(source, WatchErrorCodes.WATCH_GETTER)
        : (source as () => any)
    } else {
      // no cb -> simple effect
      getter = () => {
        if (cleanup) {
          pauseTracking()
          try {
            cleanup()
          } finally {
            resetTracking()
          }
        }
        const currentEffect = activeWatcher
        activeWatcher = effect
        try {
          return call
            ? call(source, WatchErrorCodes.WATCH_CALLBACK, [boundCleanup])
            : source(boundCleanup)
        } finally {
          activeWatcher = currentEffect
        }
      }
    }
  } else {
    getter = NOOP
    __DEV__ && warnInvalidSource(source)
  }
  //...
  // 为watch创建ReactiveEffect实例，每个watch对应一个ReactiveEffect实例
  effect = new ReactiveEffect(getter)
  // 定义effect的scheduler 最终调用callback 这个函数会在派发更新时调用
  effect.scheduler = scheduler
    ? () => scheduler(job, false)
    : (job as EffectScheduler)
  // ...
}

// /core/packages/reactivity/src/computed.ts
// computed类型的effect定义
// 从源代码中看Computed是对Subscriber的实现，effect也指向了Computed自身
// fn函数就是在调用computed所传入的参数函数，则callback则是dep上所收集到的subscriber
// 3. computed类型的effect定义
export class ComputedRefImpl<T = any> implements Subscriber {
  /**
   * @internal
   */
  _value: any = undefined
  /**
   * @internal
   */
  readonly dep: Dep = new Dep(this)
  //...
  // for backwards compat
  effect: this = this
  // ...
  constructor(
    public fn: ComputedGetter<T>,
    private readonly setter: ComputedSetter<T> | undefined,
    isSSR: boolean,
  ) {
    this[ReactiveFlags.IS_READONLY] = !setter
    this.isSSR = isSSR
  }
  // ...
}
```
![image](https://origin.picgo.net/2025/09/06/image56d8f42930c69d90.png)