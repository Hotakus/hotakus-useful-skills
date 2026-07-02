---
name: typescript-guidelines
description: TypeScript 编码规范——写 TypeScript 代码时遵循的类型系统使用、strict 配置基线、模块组织、命名约定和编码准则。适用于 TS 库、Web 前后端、Node.js 服务端的 TypeScript 编码场景。
license: MIT
metadata:
  tags: typescript, coding-standards, type-safety, strict-mode, module-organization
---

# TypeScript 编码规范

TypeScript 的类型系统是双刃剑：合理使用可以消除整类运行时错误，滥用则会导致类型复杂度失控、编译速度下降、代码难以维护。以下规则在"类型安全"和"生产力"之间划定边界。

## 1. Strict Mode 基线

**所有新项目必须从 `strict: true` 起步，并按需追加额外严格标志。**

`strict: true` 一次性启用以下所有标志：

| 标志 | 捕获的问题 |
|------|-----------|
| `strictNullChecks` | `null`/`undefined` 被当作有效值传递 |
| `noImplicitAny` | 参数/变量被隐式推断为 `any` |
| `strictFunctionTypes` | 函数参数逆变检查 |
| `strictBindCallApply` | `.bind()/.call()/.apply()` 参数类型检查 |
| `strictPropertyInitialization` | 类属性未在构造函数中初始化 |
| `noImplicitThis` | `this` 被隐式推断为 `any` |
| `useUnknownInCatchVariables` | catch 变量类型为 `unknown` 而非 `any` |
| `alwaysStrict` | 每个输出文件都 emit `"use strict"` |
| `strictBuiltinIteratorReturn` (TS 5.6+) | 内置迭代器返回 `undefined` 而非 `any` |

**正确的最小配置（Vite/Webpack 等打包器项目）**：

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "verbatimModuleSyntax": true,
    "isolatedModules": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true,
    "target": "es2022",
    "module": "esnext",
    "moduleResolution": "bundler"
  }
}
```

> **`moduleResolution` 选型指南**（来源：[TS 官方文档](https://www.typescriptlang.org/tsconfig/#moduleResolution)）：
>
> | 项目类型 | 推荐值 | 理由 |
> |---------|--------|------|
> | Vite / Webpack / Next.js | `"bundler"` | 打包器自行处理模块解析，无需 `.js` 扩展名 |
> | Node.js ESM（原生） | `"nodenext"` | 须在 import 中写 `.js` 扩展名（Node 约束） |
> | Node.js CJS | `"node16"` | 传统 CommonJS 模块 |
>
> 大部分 Tauri 前端项目使用 Vite，应选择 `"bundler"` 而非 `"nodenext"`。

**正确的最小配置（Node.js 原生 ESM 项目）**：

```json
{
  "compilerOptions": {
    "strict": true,
    "target": "es2022",
    "module": "nodenext",
    "moduleResolution": "nodenext"
  }
}
```

**错误**：仅开启 `strict: true` 而不追加 `noUncheckedIndexedAccess`。

```typescript
const arr: string[] = [];
const x: string = arr[0]; // 运行时 Crash：arr[0] 实际是 undefined
```

**正确**：启用 `noUncheckedIndexedAccess` 后必须处理 undefined：

```typescript
const arr: string[] = [];
const x = arr[0]; // 类型为 string | undefined
if (x !== undefined) {
  console.log(x.length); // 安全
}
```

## 2. 模块与导入导出

**使用 `verbatimModuleSyntax` 强制区分类型导入和值导入。**

不带 `type` 修饰符的导入保留在输出中，带 `type` 修饰符的导入被完全删除。

**正确**：

```typescript
import { useState } from "react";            // 运行时保留
import type { ReactNode } from "react";      // 编译时删除
import { type FC, type PropsWithChildren } from "react"; // 仅删除类型
import { useState, type FC } from "react";   // 混合导入合法：useState 保留，FC 删除
```

**错误**：

```typescript
import { useState, FC } from "react";   // 未标 type，verbatimModuleSyntax 下 FC 被保留到运行时输出
```

**命名空间禁令**：禁止使用 `namespace` 和 `module` 关键字组织代码。禁止使用 `import X = require(...)` 语法——统一使用 ESM `import`/`export`。

## 3. 类型系统使用规范

### 3.1 基础类型

始终使用小写原始类型：`number`、`string`、`boolean`、`symbol`、`bigint`。禁止使用装箱类型：`Number`、`String`、`Boolean`、`Symbol`、`Object`。

**正确**：

```typescript
function greet(name: string): string {
  return `Hello, ${name}`;
}
```

**错误**：

```typescript
function greet(name: String): String {
  return `Hello, ${name}`;
}
// String 是装箱类型，name 会是 string | String，行为不可预测
```

### 3.2 `any` 的使用边界

**禁止在生产代码中使用 `any`**。需要"任意类型"时使用 `unknown`，使用时必须通过类型守卫窄化。禁止使用 `// @ts-ignore`。需要抑制类型错误时使用 `// @ts-expect-error` 并附理由注释——错误消失时它会主动报错，防止过时抑制注释残留；`@ts-ignore` 不会在错误消失时报错，禁止使用。

**正确**：

```typescript
function parseJSON(input: string): unknown {
  return JSON.parse(input);
}

const data = parseJSON('{"a":1}');
if (typeof data === "object" && data !== null && "a" in data) {
  console.log((data as { a: number }).a);
}
```

**错误**：

```typescript
function parseJSON(input: string): any {
  return JSON.parse(input);
}
const data = parseJSON('{"a":1}');
console.log(data.a); // 无类型检查，运行时可能 crash
```

### 3.3 接口 vs 类型别名

**优先使用 `interface` 定义对象形状，使用 `type` 定义联合类型、交叉类型和工具类型。**

理由：`interface` 在错误信息中显示名称、支持声明合并、`extends` 性能优于 intersection。

**正确**：

```typescript
interface User {
  id: string;
  name: string;
  email: string;
}

type Status = "idle" | "loading" | "success" | "error";
type ApiResponse<T> = { data: T; error: null } | { data: null; error: string };
```

**错误**：

```typescript
type User = {
  // 能用 interface 却用了 type，失去声明合并能力和更清晰的错误信息
  id: string;
  name: string;
};
```

### 3.4 泛型使用原则

**不要定义未使用其类型参数的泛型。不要嵌套条件类型超过 2 层。**

**正确**：

```typescript
function identity<T>(value: T): T {
  return value;
}
```

**错误**：

```typescript
function print<T>(value: string): void {
  console.log(value);
}
// T 未出现在任何签名位置，是无意义的泛型参数
```

复杂条件类型的替代方案：将复杂类型抽取为命名的工具类型，并添加 `/** */` 注释说明输入输出关系。

```typescript
/** 提取 Promise 的返回值类型 */
type Await<T> = T extends Promise<infer U> ? U : T;
```

### 3.5 `satisfies` 操作符

**优先使用 `satisfies` 替代双重类型标注。**

**正确**：

```typescript
const palette = {
  red: [255, 0, 0],
  green: "#00ff00",
  blue: [0, 0, 255],
} satisfies Record<string, string | number[]>;

palette.green.startsWith("#"); // 类型推断为 string，有 .startsWith()
```

### 3.6 非空断言 `!`

**禁止使用 `!` 非空断言操作符。** 用显式空值检查或类型守卫代替。

理由：`!` 在编译时抑制了 `undefined` 检查，但运行时不会真正排除 `null`/`undefined`，是隐式逃逸。

**正确**：

```typescript
const el = document.querySelector(".btn");
if (el !== null) {
  el.addEventListener("click", handleClick);
}
```

**错误**：

```typescript
const el = document.querySelector(".btn");
el!.addEventListener("click", handleClick);  // 运行时 el 仍可能为 null
```

### 3.7 `readonly` 修饰符

**不会被重新赋值的类属性和参数用 `readonly` 标记。**

理由：`readonly` 让编译器捕获意外变更，是类型安全从"值正确"延伸到"可变性正确"的环节。

**正确**：

```typescript
class Config {
  constructor(private readonly apiKey: string) {}
}

function process(data: readonly string[]) {
  // data 不可变，意图明确
}
```

**错误**：

```typescript
class Config {
  constructor(private apiKey: string) {}  // 不会被重新赋值却不标 readonly
}
```

## 4. 函数与类

### 4.1 函数声明

**优先使用函数声明（function declaration）定义具名函数，而非函数表达式。**

**正确**：

```typescript
function calculateTotal(items: number[]): number {
  return items.reduce((sum, item) => sum + item, 0);
}
```

**错误**：

```typescript
const calculateTotal = function (items: number[]): number {
  return items.reduce((sum, item) => sum + item, 0);
};
```

### 4.2 回调类型签名

返回值不使用的回调，类型标注为 `void` 而非 `any`。回调参数不要标记为可选，除非调用方确实可以不传该参数。

**正确**：

```typescript
function fetchData(url: string, onComplete: (data: unknown) => void): void {
  // onComplete 的返回值被忽略，标注为 void
}
```

**错误**：

```typescript
function fetchData(url: string, onComplete: (data: unknown) => any): void {
  // 返回 any 模糊了"返回值被忽略"的意图
}
```

### 4.3 函数重载

**能用联合类型或可选参数代替重载的，就不要写重载。重载必须按"更具体在前，更通用在后"排序。**

**正确**：

```typescript
function format(input: string): string;
function format(input: number): string;
function format(input: string | number): string {
  return String(input);
}
```

**错误**：

```typescript
function format(input: string | number): string;  // 联合（通用）在前 —— 永远先匹配它
function format(input: string): string;           // 具体签名永远不可达
function format(input: string | number): string {
  return String(input);
}
```

## 5. 枚举使用约束

**优先使用 `as const` + 联合类型替代 `enum`。**

理由：`enum` 有运行时开销和与 `erasableSyntaxOnly`（TS 5.8+）的兼容性问题。

**正确**：

```typescript
const Status = {
  Idle: "idle",
  Loading: "loading",
  Success: "success",
  Error: "error",
} as const;

type Status = (typeof Status)[keyof typeof Status];
// 类型为 "idle" | "loading" | "success" | "error"
```

**替代方案（双向映射需 enum 时）**：

```typescript
enum HttpStatus {
  Ok = 200,
  NotFound = 404,
  InternalServerError = 500,
}
// 仅在需要反向映射（如 200 → "Ok"）时使用 enum
```

## 6. 外部 IPC/RPC 调用的类型安全

> **适用场景**：前端通过 `invoke`/`ipcRenderer`/WebSocket RPC 等方式调用外部运行时时，必须为返回值定义类型接口。本节以 Tauri `invoke` 为代表示例，Electron/WebSocket 场景同理。Tauri 命令系统架构、API 路径、命名映射见 `tauri-guidelines`。

在 Tauri 等桌面框架中，前端 TypeScript 代码需要与 Rust 后端通过 `invoke` 通信。**必须为每个外部调用定义精确的返回类型。**

**正确**：

```typescript
import { invoke } from "@tauri-apps/api/core";

// 为后端返回的数据定义接口（字段名与 Rust struct 字段对应）
interface DeviceInfo {
  name: string;
  vendorId: string;
  productId: string | null;
  controllerType: string;
}

// 泛型 invoke 获取类型安全的返回
const devices = await invoke<DeviceInfo[]>("query_devices");
devices[0].name; // 类型安全
```

**错误**：

```typescript
// 无类型标注 —— 返回 any
const devices = await invoke("query_devices");
console.log(devices[0].name); // 无类型检查
```

对于事件监听，也应为 payload 定义类型：

```typescript
import { listen } from "@tauri-apps/api/event";

interface ControllerData {
  buttons: number;
  leftStick: { x: number; y: number };
}

const unlisten = await listen<ControllerData>("update_controller", (event) => {
  console.log(event.payload.buttons); // 类型安全
});
```

## 7. 类型注释密度

**让类型推断工作。不要为推断出的类型写显式标注。**

不需要写类型标注的场景：`const` 变量初始化、函数返回值（除非需要保护接口契约）、解构赋值。

**正确**：

```typescript
const name = "Alice"; // 推断为 string，不需要 :string
const numbers = [1, 2, 3]; // 推断为 number[]
```

**错误**：

```typescript
const name: string = "Alice"; // 多余的显式标注
```

需要显式标注的场景：函数参数、类的公开 API、导出函数的返回值。非导出函数的返回值在涉及类型窄化或契约约束时也应标注。

## 8. 快速参考

| 规则 | 核心要求 | 主要例外 |
|------|---------|---------|
| 使用小写原始类型 | `number` 非 `Number` | 无 |
| 禁止裸 `any` | 用 `unknown` + 窄化代替 | 仅用于 JS→TS 迁移 |
| `interface` 优先 | 定义对象形状用 `interface` | 联合类型用 `type` |
| `as const` 替代 `enum` | 联合类型替代枚举 | 需反向映射时用 `enum` |
| 函数声明优先 | `function foo(){}` 非 `const foo = ()=>{}` | 回调/箭头函数场景 |
| `satisfies` 替代双重标注 | 验证类型但不改变推断 | TS 4.9- 不支持 |

## 9. typescript-eslint 联动

tsconfig 的 `strict` 是编译器层防线，lint 层由 `typescript-eslint` 补充。推荐启用 `recommended` 配置，关键规则：

| 规则 | 作用 | 与本规范关系 |
|------|------|-------------|
| `no-explicit-any` | 禁止裸 `any` | 落实第 3.2 节 |
| `prefer-ts-expect-error` | 强制用 `@ts-expect-error` | 落实第 3.2 节 |
| `no-non-null-assertion` | 禁止 `!` | 落实第 3.6 节 |
| `no-namespace` | 禁止 `namespace` | 落实第 10 节 |
| `no-unnecessary-type-parameters` | 检测无意义泛型 | 落实第 3.4 节 |
| `consistent-type-definitions` | `interface` vs `type` 一致性 | 落实第 3.3 节 |

**来源**：[typescript-eslint recommended 配置](https://typescript-eslint.io/linting/configs/#recommended)

## 10. 禁止事项

- 禁止使用 `var` 声明变量
- 禁止使用 `namespace` 组织代码
- 禁止使用 `any` 作为生产代码类型
- 禁止定义未使用类型参数的泛型
- 禁止使用装箱类型（`String`、`Number`、`Boolean`、`Symbol`、`Object`）
- 禁止使用 `// @ts-ignore`，改用 `// @ts-expect-error` 并附理由
- 禁止嵌套条件类型超过 2 层

## 11. 自检要求

输出 TypeScript 代码或配置前，逐项检查：

- [ ] `strict: true` 是否已设置？是否按需追加了 `noUncheckedIndexedAccess`、`exactOptionalPropertyTypes`？
- [ ] 所有导入使用了 `import type` 区分类型导入（`verbatimModuleSyntax` 强制）？
- [ ] 没有使用 `any`？必要逃生处有注释说明理由？
- [ ] 联合类型替代了不必要的函数重载？
- [ ] `as const` 替代了枚举？或枚举有正当理由（双向映射需求）？
- [ ] 条件类型嵌套不超过 2 层？
- [ ] 没有使用 `namespace`、`var`、装箱类型？
- [ ] 抑制类型错误用 `// @ts-expect-error`（非 `@ts-ignore`）并附理由？错误消失后能被及时清理？
