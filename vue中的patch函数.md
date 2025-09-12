`patch`函数的作用是将组件中`render`函数返回的`VNode`转化为真实dom，最终挂载到浏览器DOM树中，具体涉及新旧虚拟DOM树的对比，diff算法等...`patch`会针对不同类型的`VNode`进行处理。
```ts
const patch: PatchFn = (
  n1,
  n2,
  container,
  anchor = null,
  parentComponent = null,
  parentSuspense = null,
  namespace = undefined,
  slotScopeIds = null,
  optimized = __DEV__ && isHmrUpdating ? false : !!n2.dynamicChildren,
) => {
  if (n1 === n2) {  // 如果新旧VNode相同则不需要更新
    return;
  }

  // patching & not same type, unmount old tree
  if (n1 && !isSameVNodeType(n1, n2)) {  // 如果新旧VNode是不同类型则直接unmount就VNode
    anchor = getNextHostNode(n1);
    unmount(n1, parentComponent, parentSuspense, true);
    n1 = null;
  }

  if (n2.patchFlag === PatchFlags.BAIL) {
    optimized = false;
    n2.dynamicChildren = null;
  }

  const { type, ref, shapeFlag } = n2;
  switch (type) {  // 处理不同类型VNode调用不同的函数
    case Text:
      processText(n1, n2, container, anchor);
      break;
    case Comment:
      processCommentNode(n1, n2, container, anchor);
      break;
    case Static:
      if (n1 == null) {
        mountStaticNode(n2, container, anchor, namespace);
      } else if (__DEV__) {
        patchStaticNode(n1, n2, container, namespace);
      }
      break;
    case Fragment:
      processFragment(
        n1,
        n2,
        container,
        anchor,
        parentComponent,
        parentSuspense,
        namespace,
        slotScopeIds,
        optimized,
      );
      break;
    default:
      if (shapeFlag & ShapeFlags.ELEMENT) {
        processElement(
          n1,
          n2,
          container,
          anchor,
          parentComponent,
          parentSuspense,
          namespace,
          slotScopeIds,
          optimized,
        );
      } else if (shapeFlag & ShapeFlags.COMPONENT) {
        processComponent(
          n1,
          n2,
          container,
          anchor,
          parentComponent,
          parentSuspense,
          namespace,
          slotScopeIds,
          optimized,
        );
      } else if (shapeFlag & ShapeFlags.TELEPORT) {
        (type as typeof TeleportImpl).process(
          n1 as TeleportVNode,
          n2 as TeleportVNode,
          container,
          anchor,
          parentComponent,
          parentSuspense,
          namespace,
          slotScopeIds,
          optimized,
          internals,
        );
      } else if (__FEATURE_SUSPENSE__ && shapeFlag & ShapeFlags.SUSPENSE) {
        (type as typeof SuspenseImpl).process(
          n1,
          n2,
          container,
          anchor,
          parentComponent,
          parentSuspense,
          namespace,
          slotScopeIds,
          optimized,
          internals,
        );
      } else if (__DEV__) {
        warn("Invalid VNode type:", type, `(${typeof type})`);
      }
  }

  // set ref
  if (ref != null && parentComponent) {
    setRef(ref, n1 && n1.ref, parentSuspense, n2 || n1, !n2);
  } else if (ref == null && n1 && n1.ref != null) {
    setRef(n1.ref, null, parentSuspense, n1, true);
  }
};

```

#### 处理不同的`VNode`类型

##### `processText`处理Text节点
vue在进行编译的过程中会将text节点编译为类型为