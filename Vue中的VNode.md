VNode即虚拟dom在vue这种数据驱动的框架中占有重要角色，当响应式数据发生修改时并不会直接修改dom，而是会修改VNode树（虚拟Dom树），再通过`patch`函数将虚拟dom转换为真实Dom再挂载到Dom树中。
#### `VNode`的思想
`VNode`的表现形式十分类似`AST`抽象语法树。
> *关于AST的解释我们可以理解为，我们平时写的代码都是以字符串的形式表现的，但是如果想要程序或者程序高效利用我们编写的代码那么就需要将**字符串**转化为格式化数据，而这个转化结果就叫做AST。*

对比`VNode`和真实Dom，虽然Dom不是字符串但是直接操作dom会十分的不便，因此我们需要将真实Dom转化为一种方便框架使用的结构，而这种转化后的结构就是`VNode`，这种转化思想就类似于AST。

#### `VNode`的定义
```ts
export interface VNode<
  HostNode = RendererNode,
  HostElement = RendererElement,
  ExtraProps = { [key: string]: any },
> {
  /**
   * @internal
   */
  __v_isVNode: true

  /**
   * @internal
   */
  [ReactiveFlags.SKIP]: true

  type: VNodeTypes
  props: (VNodeProps & ExtraProps) | null
  key: PropertyKey | null
  ref: VNodeNormalizedRef | null
  /**
   * SFC only. This is assigned on vnode creation using currentScopeId
   * which is set alongside currentRenderingInstance.
   */
  scopeId: string | null
  /**
   * SFC only. This is assigned to:
   * - Slot fragment vnodes with :slotted SFC styles.
   * - Component vnodes (during patch/hydration) so that its root node can
   *   inherit the component's slotScopeIds
   * @internal
   */
  slotScopeIds: string[] | null
  children: VNodeNormalizedChildren
  component: ComponentInternalInstance | null
  dirs: DirectiveBinding[] | null
  transition: TransitionHooks<HostElement> | null

  // DOM
  el: HostNode | null
  placeholder: HostNode | null // async component el placeholder
  anchor: HostNode | null // fragment anchor
  target: HostElement | null // teleport target
  targetStart: HostNode | null // teleport target start anchor
  targetAnchor: HostNode | null // teleport target anchor
  /**
   * number of elements contained in a static vnode
   * @internal
   */
  staticCount: number

  // suspense
  suspense: SuspenseBoundary | null
  /**
   * @internal
   */
  ssContent: VNode | null
  /**
   * @internal
   */
  ssFallback: VNode | null

  // optimization only
  shapeFlag: number
  patchFlag: number
  /**
   * @internal
   */
  dynamicProps: string[] | null
  /**
   * @internal
   */
  dynamicChildren: (VNode[] & { hasOnce?: boolean }) | null

  // application root node only
  appContext: AppContext | null

  /**
   * @internal lexical scope owner instance
   */
  ctx: ComponentInternalInstance | null

  /**
   * @internal attached by v-memo
   */
  memo?: any[]
  /**
   * @internal index for cleaning v-memo cache
   */
  cacheIndex?: number
  /**
   * @internal __COMPAT__ only
   */
  isCompatRoot?: true
  /**
   * @internal custom element interception hook
   */
  ce?: (instance: ComponentInternalInstance) => void
}
```
`VNode`中相关的类型定义
```ts
export type VNodeTypes =
  | string // html原生节点 div p a h* span ....
  | VNode  // createVNode的时候传入一个VNode e.g. <component :is="vnode"/> 
  | Component  // 组件VNode
  | typeof Text  // 文本vnode e.g. <div>123</div> 中的123就为Text vnode
  | typeof Static  // 和静态提升有关，和响应式数据无关的dom会被提升为静态vnode
  | typeof Comment  // html中的注释，在diff中会创建comment vnode帮助确定dom插入位置
  | typeof Fragment
  | typeof Teleport  
  | typeof TeleportImpl
  | typeof Suspense
  | typeof SuspenseImpl
  
// ShapeFlags  同属于VNode属性，拓展自身属性VNodeTypes同时还增加了描述children属性的描述
export enum ShapeFlags {
  ELEMENT = 1,  // html 原生节点
  FUNCTIONAL_COMPONENT = 1 << 1,
  STATEFUL_COMPONENT = 1 << 2,
  TEXT_CHILDREN = 1 << 3,
  ARRAY_CHILDREN = 1 << 4,
  SLOTS_CHILDREN = 1 << 5,
  TELEPORT = 1 << 6,
  SUSPENSE = 1 << 7,
  COMPONENT_SHOULD_KEEP_ALIVE = 1 << 8,
  COMPONENT_KEPT_ALIVE = 1 << 9,
  COMPONENT = ShapeFlags.STATEFUL_COMPONENT | ShapeFlags.FUNCTIONAL_COMPONENT,
}


// PatchFlags
export enum PatchFlags {
  /**
   * Indicates an element with dynamic textContent (children fast path)
   */
  TEXT = 1,

  /**
   * Indicates an element with dynamic class binding.
   */
  CLASS = 1 << 1,

  /**
   * Indicates an element with dynamic style
   * The compiler pre-compiles static string styles into static objects
   * + detects and hoists inline static objects
   * e.g. `style="color: red"` and `:style="{ color: 'red' }"` both get hoisted
   * as:
   * ```js
   * const style = { color: 'red' }
   * render() { return e('div', { style }) }
   * ```
   */
  STYLE = 1 << 2,

  /**
   * Indicates an element that has non-class/style dynamic props.
   * Can also be on a component that has any dynamic props (includes
   * class/style). when this flag is present, the vnode also has a dynamicProps
   * array that contains the keys of the props that may change so the runtime
   * can diff them faster (without having to worry about removed props)
   */
  PROPS = 1 << 3,

  /**
   * Indicates an element with props with dynamic keys. When keys change, a full
   * diff is always needed to remove the old key. This flag is mutually
   * exclusive with CLASS, STYLE and PROPS.
   */
  FULL_PROPS = 1 << 4,

  /**
   * Indicates an element that requires props hydration
   * (but not necessarily patching)
   * e.g. event listeners & v-bind with prop modifier
   */
  NEED_HYDRATION = 1 << 5,

  /**
   * Indicates a fragment whose children order doesn't change.
   */
  STABLE_FRAGMENT = 1 << 6,

  /**
   * Indicates a fragment with keyed or partially keyed children
   */
  KEYED_FRAGMENT = 1 << 7,

  /**
   * Indicates a fragment with unkeyed children.
   */
  UNKEYED_FRAGMENT = 1 << 8,

  /**
   * Indicates an element that only needs non-props patching, e.g. ref or
   * directives (onVnodeXXX hooks). since every patched vnode checks for refs
   * and onVnodeXXX hooks, it simply marks the vnode so that a parent block
   * will track it.
   */
  NEED_PATCH = 1 << 9,

  /**
   * Indicates a component with dynamic slots (e.g. slot that references a v-for
   * iterated value, or dynamic slot names).
   * Components with this flag are always force updated.
   */
  DYNAMIC_SLOTS = 1 << 10,

  /**
   * Indicates a fragment that was created only because the user has placed
   * comments at the root level of a template. This is a dev-only flag since
   * comments are stripped in production.
   */
  DEV_ROOT_FRAGMENT = 1 << 11,

  /**
   * SPECIAL FLAGS -------------------------------------------------------------
   * Special flags are negative integers. They are never matched against using
   * bitwise operators (bitwise matching should only happen in branches where
   * patchFlag > 0), and are mutually exclusive. When checking for a special
   * flag, simply check patchFlag === FLAG.
   */

  /**
   * Indicates a cached static vnode. This is also a hint for hydration to skip
   * the entire sub tree since static content never needs to be updated.
   */
  CACHED = -1,
  /**
   * A special flag that indicates that the diffing algorithm should bail out
   * of optimized mode. For example, on block fragments created by renderSlot()
   * when encountering non-compiler generated slots (i.e. manually written
   * render functions, which should always be fully diffed)
   * OR manually cloneVNodes
   */
  BAIL = -2,
}
```
##### `VNodeTypes`vnode类型
在进行编译的时候会将对应的模板字符串编译为`createTextVNode`、`createStaticVNode`、`createCommentVNode`和`createVNode`四种，其实前三种最终还是调用的`createVNode`
```ts

/**
 * @private
 */
export function createTextVNode(text: string = ' ', flag: number = 0): VNode {
  return createVNode(Text, null, text, flag)
}

/**
 * @private
 */
export function createStaticVNode(
  content: string,
  numberOfNodes: number,
): VNode {
  // A static vnode can contain multiple stringified elements, and the number
  // of elements is necessary for hydration.
  const vnode = createVNode(Static, null, content)
  vnode.staticCount = numberOfNodes
  return vnode
}

/**
 * @private
 */
export function createCommentVNode(
  text: string = '',
  // when used as the v-else branch, the comment node must be created as a
  // block to ensure correct updates.
  asBlock: boolean = false,  // 控制响应式数据的依赖收集开关
): VNode {
  return asBlock
    ? (openBlock(), createBlock(Comment, null, text))
    : createVNode(Comment, null, text)
}
```
##### `ShapeFlags`
描述`VNode`属性的同时又增加了对`children`属性的描述
```ts
function _createVNode(
  type: VNodeTypes | ClassComponent | typeof NULL_DYNAMIC_COMPONENT,
  props: (Data & VNodeProps) | null = null,
  children: unknown = null,
  patchFlag: number = 0,
  dynamicProps: string[] | null = null,
  isBlockNode = false,
): VNode {
  // ...

  // encode the vnode type information into a bitmap
  // 根据传入的type给shapeFlag赋值，在patch过程也会对shapeFlag
  const shapeFlag = isString(type)
    ? ShapeFlags.ELEMENT
    : __FEATURE_SUSPENSE__ && isSuspense(type)
      ? ShapeFlags.SUSPENSE
      : isTeleport(type)
        ? ShapeFlags.TELEPORT
        : isObject(type)
          ? ShapeFlags.STATEFUL_COMPONENT
          : isFunction(type)
            ? ShapeFlags.FUNCTIONAL_COMPONENT
            : 0

   // ...

  return createBaseVNode(
    type,
    props,
    children,
    patchFlag,
    dynamicProps,
    shapeFlag,
    isBlockNode,
    true,
  )
}


function createBaseVNode(
  type: VNodeTypes | ClassComponent | typeof NULL_DYNAMIC_COMPONENT,
  props: (Data & VNodeProps) | null = null,
  children: unknown = null,
  patchFlag = 0,
  dynamicProps: string[] | null = null,
  shapeFlag: number = type === Fragment ? 0 : ShapeFlags.ELEMENT,
  isBlockNode = false,
  needFullChildrenNormalization = false,
): VNode {
  const vnode = {
    // ...
  } as VNode

  if (needFullChildrenNormalization) {
    // ...
  } else if (children) {
    // compiled element vnode - if children is passed, only possible types are
    // string or Array.
    vnode.shapeFlag |= isString(children)  // shapeFlag中增加描述children的属性
      ? ShapeFlags.TEXT_CHILDREN
      : ShapeFlags.ARRAY_CHILDREN
  }
  // ...

  return vnode
}
```
#### `VNode`的创建
`VNode`生成的来源分为两种，一种是`template`字符串模板，另一种就是在编译时候会将`setup`函数的返回值也就是`render`函数编译为返回`VNode`的函数
##### `template`模板字符
在vue代码运行的时候会遇到组件中有`template`字符串的情况，这时会调用`complie`函数在运行时将`template`字符串转化为类似于`render`函数之类的方法，⚠️由于使用vue时导出的版本默认是`runtime`版本，此版本并不包含`complie`相关逻辑，因此直接使用`template`浏览器会产生如下警告，切`template`模板不会被渲染，提示换为非`runtime-only`的版本
```text
runtime-core.esm-bundler.js:51 [Vue warn]: Component provided template option but runtime compilation is not supported in this build of Vue. Configure your bundler to alias "vue" to "vue/dist/vue.esm-bundler.js". 
```
##### 编译过程创建`VNode`
除了`template`模板字符串外`VNode`的创建还可以在`SFC`或者是`tsx`文件中的`setup`函数所返回的函数，这两者经过插件编译后都会生成创建`VNode`的函数，这也是插件`@vitejs/plugin-vue-jsx`和`@vitejs/plugin-vue`主要负责的任务
```ts
// App.tsx
import { defineComponent, ref } from "vue";

export default defineComponent({
  name: "App",
  setup() {
    const one = ref(1)

    const handleClickDiv = () => {
      one.value++;
    };
    return () => (
      <div>
        <div class="box" key="1">div Node</div>
        <div onClick={handleClickDiv}>{one.value}</div>
      </div>
    );
  },
});
```
`App.tsx`经过`@vitejs/plugin-vue-jsx`插件处理后的前后变化如下
![image](https://origin.picgo.net/2025/09/09/image0bf2e42db86f749e.png)

```vue
// App.vue
<template>
  <div>
    <div class="box" key="1">div Node</div>
    <div @click="handleClickDiv">{{ one }}</div>
  </div>
</template>

<script setup>
import { ref } from 'vue'

const one = ref(1)
const handleClickDiv = () => {
  one.value++
}
</script>
```
经过`@vitejs/plugin-vue`处理后的结果如下
![image](https://origin.picgo.net/2025/09/09/image97d6971b20f11191.png)
可以看出无论是SFC还是tsx都会被编译为一个能够返回代表组件虚拟dom树的函数

#### 组件实例和VNode的关系
组件实例中有两个属性是和虚拟dom有关的，这里的`instance`代表组件实例，他们分别是`instance.vnode`和`instance.subTree`
##### `instance.vnode`
以下是`instance.vnode`的定义和在源码中的注释
```ts
export interface ComponentInternalInstance {
  // ...
  /**
   * Vnode representing this component in its parent's vdom tree
   */
  vnode: VNode
  // ...
}
```
这个`vnode`字段所存储的是代表这个组件实例的`vnode`此处的`VNode`和父级中`subTree`虚拟dom树中存在的`VNode`是一样的
```ts
// /core/packages/runtime-core/src/renderer.ts
function baseCreateRenderer(
  options: RendererOptions,
  createHydrationFns?: typeof createHydrationFunctions,
): any {
  // ...
  const mountComponent: MountComponentFn = (
    initialVNode,
    container,
    anchor,
    parentComponent,
    parentSuspense,
    namespace: ElementNamespace,
    optimized,
  ) => {
    // ...
	const instance: ComponentInternalInstance =
      compatMountInstance ||
      (initialVNode.component = createComponentInstance( // 根据VNode创建组件实例
        initialVNode,
        parentComponent,
        parentSuspense,
      ))
    // ...
  }
  // ...
}

// /core/packages/runtime-core/src/component.ts
export function createComponentInstance(
  vnode: VNode,
  parent: ComponentInternalInstance | null,
  suspense: SuspenseBoundary | null,
): ComponentInternalInstance {
  const type = vnode.type as ConcreteComponent
  // inherit parent app context - or - if root, adopt from root vnode
  const appContext =
    (parent ? parent.appContext : vnode.appContext) || emptyAppContext

  const instance: ComponentInternalInstance = {
    uid: uid++,
    vnode,
    type,
    parent,
    appContext,
    root: null!, // to be immediately set
    next: null,
    subTree: null!, // will be set synchronously right after creation
    effect: null!,
    update: null!, // will be set synchronously right after creation
    job: null!,
    scope: new EffectScope(true /* detached */),
    render: null,
    proxy: null,
    exposed: null,
    exposeProxy: null,
    withProxy: null,

    provides: parent ? parent.provides : Object.create(appContext.provides),
    ids: parent ? parent.ids : ['', 0, 0],
    accessCache: null!,
    renderCache: [],

    // local resolved assets
    components: null,
    directives: null,

    // resolved props and emits options
    propsOptions: normalizePropsOptions(type, appContext),
    emitsOptions: normalizeEmitsOptions(type, appContext),

    // emit
    emit: null!, // to be set immediately
    emitted: null,

    // props default value
    propsDefaults: EMPTY_OBJ,

    // inheritAttrs
    inheritAttrs: type.inheritAttrs,

    // state
    ctx: EMPTY_OBJ,
    data: EMPTY_OBJ,
    props: EMPTY_OBJ,
    attrs: EMPTY_OBJ,
    slots: EMPTY_OBJ,
    refs: EMPTY_OBJ,
    setupState: EMPTY_OBJ,
    setupContext: null,

    // suspense related
    suspense,
    suspenseId: suspense ? suspense.pendingId : 0,
    asyncDep: null,
    asyncResolved: false,

    // lifecycle hooks
    // not using enums here because it results in computed properties
    isMounted: false,
    isUnmounted: false,
    isDeactivated: false,
    bc: null,
    c: null,
    bm: null,
    m: null,
    bu: null,
    u: null,
    um: null,
    bum: null,
    da: null,
    a: null,
    rtg: null,
    rtc: null,
    ec: null,
    sp: null,
  }
  //...

  return instance
}
```
在上述代码中可以看出，会根据`initialVNode`创建组件实例，然后将传入的`initialVNode`再赋值给当前组件实例的`instance.vnode`字段，而这个`initialVNode`则来自于父界组件的`subTrees`虚拟dom树中
##### `instance.subTree`
这个字段是用于存储此组件所代表的虚拟dom树的，是执行`render`函数所返回的虚拟dom树
```ts
const componentUpdateFn = () => {
  if (!instance.isMounted) {
    // ...
    if (el && hydrateNode) {
      // vnode has adopted host node - perform hydration instead of mount.
      // ...
    } else {
      //...
      const subTree = (instance.subTree = renderComponentRoot(instance));
      
      patch(
        null,
        subTree,
        container,
        anchor,
        instance,
        parentSuspense,
        namespace,
      );
  
      initialVNode.el = subTree.el;
    }
    // ...
  } else {
    // ...
  }
};
```
上述代码的`renderComponentRoot`函数会调用`render`函数并把`render`函数的执行结果所返回的虚拟dom树赋值给组件的`subTree`
#### `VNode`相比真实Dom的异同
虚拟dom这个概念最早提出是react，用于解决直接操作dom渲染界面所带来的性能瓶颈，它的核心作用可以理解为将dom操作的批量打包，然后再触发浏览器的渲染和重绘制。上述批量打包机制就是`patch`方法所实现的内容。不过虚拟dom的引入会增加复杂度因为项目要维护一个额外的虚拟Dom树。