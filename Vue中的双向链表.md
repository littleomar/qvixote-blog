双向链表是在3.5版本重构完reactive模块时出现的，双向链表的引入大幅提升了Vue的性能，节省内存占用具体可参照：[Announcing Vue 3.5](https://blog.vuejs.org/posts/vue-3-5)

#### 定义
先贴一段`doubly-linked`在Vue中的定义，不过这种双向链表属于二维双向链表，会有独立的`sub`(subscriber 订阅者)链和`dep` (dependence 依赖)链
```typescript
// /core/packages/reactivity/src/dep.ts
export class Link {
  /**
   * - Before each effect run, all previous dep links' version are reset to -1
   * - During the run, a link's version is synced with the source dep on access
   * - After the run, links with version -1 (that were never used) are cleaned
   *   up
   */
  version: number

  /**
   * Pointers for doubly-linked lists
   */
  nextDep?: Link
  prevDep?: Link
  nextSub?: Link
  prevSub?: Link
  prevActiveLink?: Link

  constructor(
    public sub: Subscriber,
    public dep: Dep,
  ) {
    this.version = dep.version
    this.nextDep =
      this.prevDep =
      this.nextSub =
      this.prevSub =
      this.prevActiveLink =
        undefined
  }
}
```
#### `Link`的结构
可以从两个角度分析链表结构，**使用过程**和**LinkNode**的定义，`Link`里边包含多个`LinkNode`并通过`LinkNode`内的指针区域进行连接
- **使用过程** 分为链表本身和链表头尾指针
```typescript
// /core/packages/reactivity/src/dep.ts  dep 中的使用
export class Dep {
  ...
  
  /**
   * Doubly linked list representing the subscribing effects (tail)
   */
  subs?: Link = undefined

  ...
}
```

```typescript
// /core/packages/reactivity/src/effect.ts  在Subscriber中的使用
export interface Subscriber extends DebuggerOptions {
  ...

  /**
   * Head of the doubly linked list representing the deps
   * @internal
   */
  deps?: Link
  
  ...
}
```
- **LinkNode** 分为两个部分**数据区域**和**指针区域**
	- **数据区域** 包含`sub` `dep` `version` 三个部分
	- **指针区域** 有`sub` `dep`两个链上的前后指针 `prevDep` `nextDep` `prevSub` `nextSub`
![image](https://origin.picgo.net/2025/09/04/image3b595cba1bfd7409.png)
#### 判断是否为是同一个`LinkNode`
假如在一个`LinkNode`节点中`sub`指向的subscriber和`dep`指向的dependence都一致，则此`LinkNode`确定为同一个，所以可以计算出整个项目中的`LinkNode`总数
$$

\sum_{i, j} (\text{effect}_i \times \text{reactive}_j)

$$
#### `LinkNode`的创建
在vue中关于`Link`的创建出现在响应式数据的**依赖收集**中，这也是在整个项目中唯一一处创建`LinkNode` 的地方
```typescript
export class Dep {
...
  track(debugInfo?: DebuggerEventExtraInfo): Link | undefined {
    ...
    let link = this.activeLink
    if (link === undefined || link.sub !== activeSub) {
      link = this.activeLink = new Link(activeSub, this)

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
      ...
    }

    return link
  }
...
}
```
#### `version`的作用
在此次`effect`运行之前会执行`prepareDeps`将所有的`deps`中的`version`标记为-1，`在执行effect`的过程中每次引用`deps`都会将其中的`version`同步为此`LinkNode`中`dep`的`version`，在`effect`运行结束后会执行`cleanupDeps`将所有`version === -1``Link`作为清除目标，根据[[JS中的垃圾回收]]机制，只需要把`LinkNode`在前后节点的引用关系清除即可：
```typescript
// /core/packages/reactivity/src/effect.ts   把version标记为-1
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

// /core/packages/reactivity/src/dep.ts   同步version版本号
export class Dep {
...
  track(debugInfo?: DebuggerEventExtraInfo): Link | undefined {
    ...
    let link = this.activeLink
    if (link === undefined || link.sub !== activeSub) {
    
    } else if (link.version === -1) {
      // reused from last run - already a sub, just sync version
      link.version = this.version
      
      ...
    }

    return link
  }
...
}

// /core/packages/reactivity/src/effect.ts   清除LinkNode
function cleanupDeps(sub: Subscriber) {
  // Cleanup unsued deps
  let head
  let tail = sub.depsTail
  let link = tail
  while (link) {
    const prev = link.prevDep
    if (link.version === -1) {  // 判断version === -1 开始执行引用清除操作
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
    ...
  }
}

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
  ...
}

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
#### 性能提升
对于Vue这种数据驱动的响应式框架，频繁更新依赖是很常见的，所以3.5版本使用双向链表重构响应式模块带来的性能提升就显而易见了，在插入、删除、遍历通知操作更快，降低垃圾回收压力等。
#### 与3.5之前版本的对比
TO BE CONTINUE...