在vue中和编译相关的内容就是将`.js .ts .vue`文件中所写的和`html`字符串相关的内容例如`template`模板字符串，`setup`内的`render`函数和SFC文件中的`<template>`模板中的字符串转换为能够创建[[Vue中的VNode|VNode]]的js代码。
当然和代码编译相关的内容一般离不开**AST**（Abstract Syntax Tree，抽象语法树），直接操作字符串效率比较低，因此需要将`html`字符串先转化为格式化数据即AST，然后再做后续操作
```tree
./core/packages/compiler-core/src/
├── ast.ts
├── babelUtils.ts
├── codegen.ts
├── compat
│   ├── compatConfig.ts
│   └── transformFilter.ts
├── compile.ts
├── errors.ts
├── index.ts
├── options.ts
├── parser.ts
├── runtimeHelpers.ts
├── tokenizer.ts
├── transform.ts
├── transforms
│   ├── cacheStatic.ts
│   ├── noopDirectiveTransform.ts
│   ├── transformElement.ts
│   ├── transformExpression.ts
│   ├── transformSlotOutlet.ts
│   ├── transformText.ts
│   ├── vBind.ts
│   ├── vFor.ts
│   ├── vIf.ts
│   ├── vMemo.ts
│   ├── vModel.ts
│   ├── vOn.ts
│   ├── vOnce.ts
│   └── vSlot.ts
├── utils.ts
└── validateExpression.ts
```
#### `template`模板字符串
vue通过调用`compile`函数将模板字符串转为render函数，这里解析`template`模板字符串时使用的是[htmlparse2](https://github.com/fb55/htmlparser2/tree/masterhttps://github.com/fb55/htmlparser2/tree/master)
```ts
// /core/packages/runtime-core/src/component.ts
export function finishComponentSetup(
  instance: ComponentInternalInstance,
  isSSR: boolean,
  skipOptions?: boolean,
): void {
  // ...
  // 此时检测到没有render函数的情况下会把template模板编译为render函数
  // template / render function normalization
  // could be already set when returned from setup()
  if (!instance.render) {
    // ...
    Component.render = compile(template, finalCompilerOptions)
    // ...
  }
  // ...
  // 如果在runtime-only版本使用template模板字符串则会报一下错误
  // warn missing template/render
  // the runtime compilation of template in SSR is done by server-render
  if (__DEV__ && !Component.render && instance.render === NOOP && !isSSR) {
    if (!compile && Component.template) {
      /* v8 ignore start */
      warn(
        `Component provided template option but ` +
          `runtime compilation is not supported in this build of Vue.` +
          (__ESM_BUNDLER__
            ? ` Configure your bundler to alias "vue" to "vue/dist/vue.esm-bundler.js".`
            : __ESM_BROWSER__
              ? ` Use "vue.esm-browser.js" instead.`
              : __GLOBAL__
                ? ` Use "vue.global.js" instead.`
                : ``) /* should not happen */,
      )
      /* v8 ignore stop */
    } else {
      warn(`Component is missing template or render function: `, Component)
    }
  }
}
```
![image](https://origin.picgo.net/2025/09/09/imaged8780de6421f0007.png)
##### parse
这个过程主要调用开源的html解析工具[htmlparse2](https://github.com/fb55/htmlparser2/tree/masterhttps://github.com/fb55/htmlparser2/tree/master)，借用里边高效html解析工具将其解析为AST，不过这个AST只是初步解析并将`vue`相关的关键字用不同的`name`字段在AST节点中表示
```ts
// <div v-for="i in list">{{ i }}</div>
// for循环会被加上 name: 'for'
{
  type: 7,
  name: "for",
  rawName: "v-for",
  exp: {
    type: 4,
    loc: {
      start: { column: 17, line: 5, offset: 141 },
      end: { column: 26, line: 5, offset: 150 },
      source: "i in list",
    },
    content: "i in list",
    isStatic: false,
    constType: 0,
  },
  modifiers: [],
  loc: {
    start: { column: 10, line: 5, offset: 134 },
    end: { column: 27, line: 5, offset: 151 },
    source: 'v-for="i in list"',
  },
  forParseResult: {
    // ...
  },
}

//  <div v-if="ifShow">{{ `if-${ifShow}` }}</div>
// if判断会被加上 name: 'if' 的属性
{
  type: 1,
  tag: "div",
  ns: 0,
  tagType: 0,
  props: [
    {
      type: 7,
      name: "if",
      rawName: "v-if",
      exp: {
        type: 4,
        loc: {
          start: { column: 16, line: 3, offset: 65 },
          end: { column: 22, line: 3, offset: 71 },
          source: "ifShow",
        },
        content: "ifShow",
        isStatic: false,
        constType: 0,
      },
      modifiers: [],
      loc: {
        start: { column: 10, line: 3, offset: 59 },
        end: { column: 23, line: 3, offset: 72 },
        source: 'v-if="ifShow"',
      },
    },
  ],
  children: [
    {
      type: 5,
      content: {
        type: 4,
        loc: {
          start: { column: 27, line: 3, offset: 76 },
          end: { column: 31, line: 3, offset: 80 },
          source: "'if'",
        },
        content: "'if'",
        isStatic: false,
        constType: 0,
      },
      loc: {
        start: { column: 24, line: 3, offset: 73 },
        end: { column: 34, line: 3, offset: 83 },
        source: "{{ 'if' }}",
      },
    },
  ],
  loc: {
    start: { column: 5, line: 3, offset: 54 },
    end: { column: 40, line: 3, offset: 89 },
    source: "<div v-if=\"ifShow\">{{ 'if' }}</div>",
  },
}

// <div @click="handleClickDiv">{{ one }}</div>
// '@' 绑定事件的函数会被加上 name: 'on'属性
{
  type: 7,
  name: "on",
  rawName: "@click",
  exp: {
    type: 4,
    loc: {
      start: { column: 18, line: 6, offset: 192 },
      end: { column: 32, line: 6, offset: 206 },
      source: "handleClickDiv",
    },
    content: "handleClickDiv",
    isStatic: false,
    constType: 0,
  },
  arg: {
    type: 4,
    loc: {
      start: { column: 11, line: 6, offset: 185 },
      end: { column: 16, line: 6, offset: 190 },
      source: "click",
    },
    content: "click",
    isStatic: true,
    constType: 3,
  },
  modifiers: [],
  loc: {
    start: { column: 10, line: 6, offset: 184 },
    end: { column: 33, line: 6, offset: 207 },
    source: '@click="handleClickDiv"',
  },
}
```
##### transform 
接收上一步`parse`函数生成的AST，并对AST进行进一步加工，加工内容主要都存储在其参数中
```ts
// we name it `baseCompile` so that higher order compilers like
// @vue/compiler-dom can export `compile` while re-exporting everything else.
export function baseCompile(
  source: string | RootNode,
  options: CompilerOptions = {},
): CodegenResult {
  // ...

  transform(
    ast,
    extend({}, resolvedOptions, {
      nodeTransforms: [
        ...nodeTransforms,
        ...(options.nodeTransforms || []), // user transforms
      ],
      directiveTransforms: extend(
        {},
        directiveTransforms,
        options.directiveTransforms || {}, // user transforms
      ),
    }),
  )

  // ...
}
```
调用transform的时候会传入参数，其中参数属性主要有两个`nodeTransforms`和`directiveTransforms`主要是对`template`模板字符串中的*节点* 和*指令* 的处理，其他属性则是一些工具函数，以下是它们的定义和解释
```ts
// There are two types of transforms:
//
// - NodeTransform:
//   Transforms that operate directly on a ChildNode. NodeTransforms may mutate,
//   replace or remove the node being processed.
export type NodeTransform = (
  node: RootNode | TemplateChildNode,
  context: TransformContext,
) => void | (() => void) | (() => void)[]

// - DirectiveTransform:
//   Transforms that handles a single directive attribute on an element.
//   It translates the raw directive into actual props for the VNode.
export type DirectiveTransform = (
  dir: DirectiveNode,
  node: ElementNode,
  context: TransformContext,
  // a platform specific compiler can import the base transform and augment
  // it by passing in this optional argument.
  augmentor?: (ret: DirectiveTransformResult) => DirectiveTransformResult,
) => DirectiveTransformResult
```
- `nodeTransforms`加工AST中的节点的方法，对照着`VNode`中的`PatchFlags`，判断出节点对应哪种`PatchFlags`，比如是需要`createTextVNode`、`createCommentVNode`、`createElementVNode`等都是在这个过程加工的，从宏观角度看这个过程就是处理dom的
```ts
// /core/packages/compiler-core/src/transforms/transformText.ts
// 例如在compiler模块的transforms转换方法下transformText的作用就是将判断为TEXT的节点组成带有`createTextVNode`的函数

// Merge adjacent text nodes and expressions into a single expression
// e.g. <div>abc {{ d }} {{ e }}</div> should have a single expression node as child.
export const transformText: NodeTransform = (node, context) => {
  if (
    node.type === NodeTypes.ROOT ||
    node.type === NodeTypes.ELEMENT ||
    node.type === NodeTypes.FOR ||
    node.type === NodeTypes.IF_BRANCH
  ) {
    // perform the transform on node exit so that all expressions have already
    // been processed.
    return () => {
      const children = node.children
      let currentContainer: CompoundExpressionNode | undefined = undefined
      let hasText = false

      for (let i = 0; i < children.length; i++) {
        const child = children[i]
        if (isText(child)) {
          hasText = true
          for (let j = i + 1; j < children.length; j++) {
            const next = children[j]
            if (isText(next)) {
              if (!currentContainer) {
                currentContainer = children[i] = createCompoundExpression(
                  [child],
                  child.loc,
                )
              }
              // merge adjacent text node into current
              currentContainer.children.push(` + `, next)
              children.splice(j, 1)
              j--
            } else {
              currentContainer = undefined
              break
            }
          }
        }
      }

      if (
        !hasText ||
        // if this is a plain element with a single text child, leave it
        // as-is since the runtime has dedicated fast path for this by directly
        // setting textContent of the element.
        // for component root it's always normalized anyway.
        (children.length === 1 &&
          (node.type === NodeTypes.ROOT ||
            (node.type === NodeTypes.ELEMENT &&
              node.tagType === ElementTypes.ELEMENT &&
              // #3756
              // custom directives can potentially add DOM elements arbitrarily,
              // we need to avoid setting textContent of the element at runtime
              // to avoid accidentally overwriting the DOM elements added
              // by the user through custom directives.
              !node.props.find(
                p =>
                  p.type === NodeTypes.DIRECTIVE &&
                  !context.directiveTransforms[p.name],
              ) &&
              // in compat mode, <template> tags with no special directives
              // will be rendered as a fragment so its children must be
              // converted into vnodes.
              !(__COMPAT__ && node.tag === 'template'))))
      ) {
        return
      }

      // pre-convert text nodes into createTextVNode(text) calls to avoid
      // runtime normalization.
      for (let i = 0; i < children.length; i++) {
        const child = children[i]
        if (isText(child) || child.type === NodeTypes.COMPOUND_EXPRESSION) {
          const callArgs: CallExpression['arguments'] = []
          // createTextVNode defaults to single whitespace, so if it is a
          // single space the code could be an empty call to save bytes.
          if (child.type !== NodeTypes.TEXT || child.content !== ' ') {
            callArgs.push(child)
          }
          // mark dynamic text with flag so it gets patched inside a block
          if (
            !context.ssr &&
            getConstantType(child, context) === ConstantTypes.NOT_CONSTANT
          ) {
            callArgs.push(
              PatchFlags.TEXT +
                (__DEV__ ? ` /* ${PatchFlagNames[PatchFlags.TEXT]} */` : ``),
            )
          }
          children[i] = {
            type: NodeTypes.TEXT_CALL,
            content: child,
            loc: child.loc,
            codegenNode: createCallExpression(  // 此时会转换为createTextVNode字符串
              context.helper(CREATE_TEXT), // 这个helper就是关系映射 'CREATE_TEXT -> createTextVNode'
              callArgs,
            ),
          }
        }
      }
    }
  }
}
```
- `directiveTransforms` 处理vue的相关指令的转换方法将原生的指令转换为实际的VNode属性，比如将for循环转换为`list.map`函数，宏观角度看这个过程是处理js的
##### generate
根据已经生成的完整的AST抽象语法树，转换为渲染函数字符串

#### `@vitejs/plugin-vue`插件处理
这个插件是依赖vue编译模块的，会调用`vue/compiler-sfc`其实执行过程和`template`执行很类似，不过执行时机不同，插件是在静态资源构建时运行的，`template`模板则是在`runtime`时候执行
#### `@vitejs/plugin-vue-jsx`插件处理
这个插件就与前两者不同了，因为处理的文件是`jsx`或者`tsx`，而他们又直接被`babel`支持所以此插件的本质是调用`babel` + `babel-plugin`。此插件并不依赖vue中的编译模块，而是把和vue编译相关的逻辑转移到了`babel-plugin`之中，*使用`babel`编译的模板中是不包含`patchFlags`优化属性的*
![image](https://origin.picgo.net/2025/09/11/imagef67a55dc820b0190.png)
#### 通过`complier`模块和`tsx+babel`编译的区别

|              | `complier`                                        | `tsx+babel`             |
| ------------ | ------------------------------------------------- | ----------------------- |
| 静态标识和静态提升    | 会标记出静态节点优化，优化diff算法                               | ❌                       |
| `PatchFlags` | 会在render函数中加入PatchFlags参数，优化diff算法                | ❌                       |
| `block tree` |                                                   |                         |
| 性能影响         | 加入了许多优化diff算法的参数，所以提高了runtime时的性能，不过构建时候的市场也会响应增加 | runtime ☹️<br>build  😄 |
