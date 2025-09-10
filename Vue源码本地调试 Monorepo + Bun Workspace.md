
### 初始化测试项目

```bash
mkdir vue-debugger
bun init
```

###  下载vue源码

直接在项目根目录内down下vue3的源码

```bash
git clone https://github.com/vuejs/core.git  # 下载vue源码
```

### 配置package.json

将vue包配置为本地环境，package.json中添加workspace相关配置，即不会从npm源进行下载。

```diff
{
  "name": "vue-debugger",
  "module": "index.ts",
  "type": "module",
  "private": true,
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
+ "workspaces": {
+   "packages": [
+     "core/packages/*"
+   ]
+  },
  "devDependencies": {
    "@types/bun": "latest",
  },
  "peerDependencies": {
    "typescript": "^5"
  },
+ "dependencies": {
+   "vue": "workspace:*"
+ }
}

```

顺便把debugger项目所需要的包也装上：

- @vitejs/plugin-vue      // 解析vue sfc文件
- @vitejs/plugin-vue-jsx   // 支持tsx文件解析
- vite
- vite-plugin-inspect   // vite 插件监控插件，可以清除看到插件对资源所做的事情，处理前后的对比

### 配置为源码调试

虽然vue包已经按照本地加载的方式进行处理，但是默认会加载 `core/packages` 中构建后的产物，如果想直接调用vue的源码，需要对`vite.config.ts`做如下配置，添加如下 `alias`中的内容

```tsx
import vueJsx from "@vitejs/plugin-vue-jsx";
import { defineConfig } from "vite";
import path from "path";
import vue from "@vitejs/plugin-vue";
import Inspect from "vite-plugin-inspect";

const pathSrc = path.resolve(__dirname, "src");
const resolvePath = (p: string) =>
  path.resolve(__dirname, `./core/packages/${p}/src/index.ts`);

export default () => {
  return defineConfig({
    plugins: [Inspect(), vueJsx({}), vue({})],
    resolve: {
      alias: {
        "@/": `${pathSrc}/`,
        vue: path.resolve(__dirname, `./core/packages/vue/src/index.ts`),
        "@vue/runtime-core": resolvePath("runtime-core"),
        "@vue/runtime-dom": resolvePath("runtime-dom"),
        "@vue/reactivity": resolvePath("reactivity"),
        "@vue/shared": resolvePath("shared"),
        "@vue/compiler-dom": resolvePath("compiler-dom"),
        "@vue/compiler-core": resolvePath("compiler-core"),
        "@vue/compiler-ssr": resolvePath("compiler-ssr"),
        "@vue/template-explorer": resolvePath("template-explorer"),
        "@vue/server-renderer": resolvePath("server-renderer"),
        "@vue/devtools-api": resolvePath("devtools-api"),
      },
    }
  });
};
```

### 配置全局变量

我们平时用的vue都是构建后的，然而在vue代码构建的过程中需要用到许多全局变量，如果我们直接使用源码进行调试会发现这些全局变量是缺失的，我们可以根据缺失的变量在 `core` 中进行搜索，然后在 `vite.config.ts` 的 `define` 中定义一下即可，以下是完整配置。

```tsx
import vueJsx from "@vitejs/plugin-vue-jsx";
import { defineConfig } from "vite";
import path from "path";
import vue from "@vitejs/plugin-vue";
import Inspect from "vite-plugin-inspect";

const pathSrc = path.resolve(__dirname, "src");
const resolvePath = (p: string) =>
  path.resolve(__dirname, `./core/packages/${p}/src/index.ts`);

export default () => {
  return defineConfig({
    plugins: [Inspect(), vueJsx({}), vue({})],
    resolve: {
      alias: {
        "@/": `${pathSrc}/`,
        vue: path.resolve(__dirname, `./core/packages/vue/src/index.ts`),
        "@vue/runtime-core": resolvePath("runtime-core"),
        "@vue/runtime-dom": resolvePath("runtime-dom"),
        "@vue/reactivity": resolvePath("reactivity"),
        "@vue/shared": resolvePath("shared"),
        "@vue/compiler-dom": resolvePath("compiler-dom"),
        "@vue/compiler-core": resolvePath("compiler-core"),
        "@vue/compiler-ssr": resolvePath("compiler-ssr"),
        "@vue/template-explorer": resolvePath("template-explorer"),
        "@vue/server-renderer": resolvePath("server-renderer"),
        "@vue/devtools-api": resolvePath("devtools-api"),
      },
    },
    css: {
      modules: {
        localsConvention: "camelCase",
      },
    },
    define: {
      __DEV__: true,
      __BROWSER__: true,
      __TEST__: false,
      __ESM_BUNDLER__: true,
      __COMPAT__: false,
      __FEATURE_OPTIONS_API__: true,
      __FEATURE_SUSPENSE__: true,
      __FEATURE_PROD_DEVTOOLS__: false,
      __SSR__: false,
      __VERSION__: JSON.stringify("3.5.16"),
      __VUE_PROD_HYDRATION_MISMATCH_DETAILS__: false,
    },
  });
};
```

### 运行项目

```bash
bun run --filter "debug" dev
```

### 附：vscode中对vue代码打断点调试

#### 在项目根目录的`.vscode` 目录下新建文件`launch.json`

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "chrome",
      "request": "launch",
      "name": "Launch Chrome with Vue Debugger",
      "url": "http://localhost:5173",
      "webRoot": "${workspaceFolder}",  // 为 /core 和 /debugger 的父级目录 
      "sourceMapPathOverrides": {
        "http://localhost:5173/*": "${workspaceFolder}/debugger/*"
      }
    }
  ]
}
```
#### 配置 `tsconfig.json` 适配 Cmd+click 跳转到 src 源码
```diff
{
  "compilerOptions": {
    // Environment setup & latest features
    "lib": ["ESNext"],
    "target": "ESNext",
    "module": "ESNext",
    "moduleDetection": "force",
    "jsx": "react",
    "allowJs": true,

    // Bundler mode
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "verbatimModuleSyntax": true,
    "noEmit": true,

    // Path mapping for Vue source code
    "baseUrl": ".",
+    "paths": {
+      "vue": ["./core/packages/vue/src/index.ts"],
+      "vue/*": ["./core/packages/vue/src/*"],
+      "@vue/shared": ["./core/packages/shared/src/index.ts"],
+      "@vue/reactivity": ["./core/packages/reactivity/src/index.ts"],
+      "@vue/runtime-core": ["./core/packages/runtime-core/src/index.ts"],
+      "@vue/runtime-dom": ["./core/packages/runtime-dom/src/index.ts"],
+      "@vue/compiler-core": ["./core/packages/compiler-core/src/index.ts"],
+      "@vue/compiler-dom": ["./core/packages/compiler-dom/src/index.ts"],
+      "@vue/compiler-sfc": ["./core/packages/compiler-sfc/src/index.ts"],
+      "@vue/compiler-ssr": ["./core/packages/compiler-ssr/src/index.ts"],
+      "@vue/server-renderer": ["./core/packages/server-renderer/src/index.ts"]
+    },

    // Best practices
    "strict": true,
    "skipLibCheck": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,

    // Some stricter flags (disabled by default)
    "noUnusedLocals": false,
    "noUnusedParameters": false,
    "noPropertyAccessFromIndexSignature": false
  }
}
```