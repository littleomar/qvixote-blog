Vue中的响应式数据发生修改时会对[[learn vue/Vue响应式的依赖收集|依赖收集]]创建的`Dep.subs`链路上的订阅者发起更新通知，触发`Dep.notify()`函数进行通知
```typescript
export class Dep {
  version = 0
  /**
   * Doubly linked list representing the subscribing effects (tail)
   */
  subs?: Link = undefined
  
  // ...

  notify(debugInfo?: DebuggerEventExtraInfo): void {
    startBatch()
    try {
      for (let link = this.subs; link; link = link.prevSub) {
        if (link.sub.notify()) {
          // if notify() returns `true`, this is a computed. Also call notify
          // on its dep - it's called here instead of inside computed's notify
          // in order to reduce call stack depth.
          ;(link.sub as ComputedRefImpl).dep.notify()
        }
      }
    } finally {
      endBatch()
    }
  }
}

// /core/packages/reactivity/src/effect.ts
// effect中的notify，上边派发更新时会调用这个方法
export class ReactiveEffect<T = any>
  implements Subscriber, ReactiveEffectOptions
{

  /**
   * @internal
   */
  notify(): void {
    if (
      this.flags & EffectFlags.RUNNING &&
      !(this.flags & EffectFlags.ALLOW_RECURSE)
    ) {
      return
    }
    if (!(this.flags & EffectFlags.NOTIFIED)) {
      batch(this)
    }
  }
}
```
 此时会开启一个`startBatch`批处理的函数，具体内容稍后再看，遍历响应式数据`subs`链，调用`LinkNode`中的`sub.notify()`完成通知操作，如果此`sub`为`computed`则会进一步调用`dep.notify()`，此处的`notify`最终会通知到`effect.callback`不过可能会在调用前加一层调度函数
####  `startBatch & endBatch`的作用
在`effect.ts`中会存储一个公共的`batchDepth: number`的层级计数器，默认为`0`，每次调用`startBatch`这个层级就会`+1`，在`endBatch`时如果`batchDepth`层级大于`0`就会直接跳过，`batch`的作用是创建一个待处理的`sub`链表，这更像是平铺操作，把所有的代更新`sub`收集起来组成一个链表然后统一处理
```typescript
// /core/packages/reactivity/src/effect.ts
let batchDepth = 0
let batchedSub: Subscriber | undefined
let batchedComputed: Subscriber | undefined

export function batch(sub: Subscriber, isComputed = false): void {
  sub.flags |= EffectFlags.NOTIFIED
  if (isComputed) {
    sub.next = batchedComputed
    batchedComputed = sub
    return
  }
  sub.next = batchedSub
  batchedSub = sub
}

export function startBatch(): void {
  batchDepth++
}

/**
 * Run batched effects when all batches have ended
 * @internal
 */
export function endBatch(): void {
  if (--batchDepth > 0) {
    return
  }

  if (batchedComputed) {
    let e: Subscriber | undefined = batchedComputed
    batchedComputed = undefined
    while (e) {
      const next: Subscriber | undefined = e.next
      e.next = undefined
      e.flags &= ~EffectFlags.NOTIFIED
      e = next
    }
  }

  let error: unknown
  while (batchedSub) {
    let e: Subscriber | undefined = batchedSub
    batchedSub = undefined
    while (e) {
      const next: Subscriber | undefined = e.next
      e.next = undefined
      e.flags &= ~EffectFlags.NOTIFIED
      if (e.flags & EffectFlags.ACTIVE) {
        try {
          // ACTIVE flag is effect-only
          ;(e as ReactiveEffect).trigger()
        } catch (err) {
          if (!error) error = err
        }
      }
      e = next
    }
  }

  if (error) throw error
}
```
`startBatch`和`endBatch`一定是成对出现的，只有这样才能正确控制`batchDepth`的层级

从源代码全局搜索中可以看到`startBatch`和`endBatch`有三个使用场景
##### `Dep.notify`
此时会在通知时开启一个层级，假如为`ref`响应式数据，此时`batchDepth`层级不会增加，假如为`computed`则层级会进一步增加，待`computed.dep.subs`都通知完成后会退出`computed`层级，因此`computed`类型的`ref`和`computed`内的`subs`为同一层级
```typescript
// /core/packages/reactivity/src/dep.ts
// Dep.notify()
  notify(debugInfo?: DebuggerEventExtraInfo): void {
    startBatch()
    try {
      if (__DEV__) {
        // DEV...
      }
      for (let link = this.subs; link; link = link.prevSub) {
        if (link.sub.notify()) {
          // if notify() returns `true`, this is a computed. Also call notify
          // on its dep - it's called here instead of inside computed's notify
          // in order to reduce call stack depth.
          ;(link.sub as ComputedRefImpl).dep.notify()
        }
      }
    } finally {
      endBatch()
    }
  }
```
#### `trigger() -> dep.trigger()`
~~在`reactive`类型的响应式数据会调用`trigger`方法创建一个层级然后把所有待更新的子项`sub`平铺到`batchedSub`链表中~~
```typescript
// /core/packages/reactivity/src/baseHandlers.ts
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
    let oldValue = target[key]
    if (!this._isShallow) {
      const isOldValueReadonly = isReadonly(oldValue)
      if (!isShallow(value) && !isReadonly(value)) {
        oldValue = toRaw(oldValue)
        value = toRaw(value)
      }
      if (!isArray(target) && isRef(oldValue) && !isRef(value)) {
        if (isOldValueReadonly) {
          if (__DEV__) {
            warn(
              `Set operation on key "${String(key)}" failed: target is readonly.`,
              target[key],
            )
          }
          return true
        } else {
          oldValue.value = value
          return true
        }
      }
    } else {
      // in shallow mode, objects are set as-is regardless of reactive or not
    }

    const hadKey =
      isArray(target) && isIntegerKey(key)
        ? Number(key) < target.length
        : hasOwn(target, key)
    const result = Reflect.set(
      target,
      key,
      value,
      isRef(target) ? target : receiver,
    )
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

// 数组中的某些方法在调用时会开启一个层级
// /core/packages/reactivity/src/arrayInstrumentations.ts
// instrument length-altering mutation methods to avoid length being tracked
// which leads to infinite loops in some cases (#2137)
function noTracking(
  self: unknown[],
  method: keyof Array<any>,
  args: unknown[] = [],
) {
  pauseTracking()
  startBatch()
  const res = (toRaw(self) as any)[method].apply(self, args)
  endBatch()
  resetTracking()
  return res
}
```