---
name: react-guidelines
description: React 19 前端开发规范——写 React 组件（Server/Client Components）、Hooks、状态管理、项目结构时的编码准则。覆盖组件模式、Server Components vs Client Components 边界、表单与 Actions、TypeScript 集成。
license: MIT
metadata:
  tags: react, frontend, components, hooks, server-components, typescript
---

# React 19 前端开发规范

React 19（2024 年 12 月稳定版）引入了 React Compiler、Actions、`use()` API、Server Components（框架集成）等范式级变更。手动 `useMemo`/`useCallback` 不再是日常必需品，表单处理被 Actions 简化，数据加载从 `useEffect` 迁移到 Server Components 和 Suspense。以下规则基于 React 19 + TypeScript 5.x 编写。

## 1. 组件架构：Server First

**在支持 RSC 的框架（Next.js 15+、Remix、Waku）中，文件默认是 Server Component。仅在需要交互性时才添加 `"use client"`。**

Server Component 可以：
- 直接访问数据库和私有 API
- 不向浏览器发送 JavaScript
- 使用 `async/await` 直接渲染

Client Component 需要 `"use client"` 当且仅当：
- 使用 `useState`、`useEffect`、`useReducer`、refs、浏览器 API
- 使用事件处理器（`onClick`、`onChange` 等）
- 使用第三方客户端库（图表、编辑器、地图）

**正确**：

```tsx
// app/page.tsx —— Server Component
import { getPosts } from "@/lib/db";
import { PostList } from "@/components/post-list";

export default async function Page() {
  const posts = await getPosts();
  return <PostList posts={posts} />;
}
```

```tsx
// components/post-list.tsx —— Client Component（因为有交互）
"use client";

import { useState } from "react";

export function PostList({ posts }: { posts: Post[] }) {
  const [search, setSearch] = useState("");
  // ...
}
```

**错误**：在 Server Component 中直接使用 hooks 或事件处理器。

```tsx
// 这个文件没有 "use client" 但使用了 hooks —— 运行时错误
export function Counter() {
  const [count, setCount] = useState(0); // Server Component 不可用 useState
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

## 2. 组合优于配置

**用 `children` 和命名 slot 实现布局灵活性，不要用 10+ 个 boolean props 控制组件变体。**

**正确**：

```tsx
interface CardProps {
  title: string;
  children: React.ReactNode;
  footer?: React.ReactNode;
}

export function Card({ title, children, footer }: CardProps) {
  return (
    <section>
      <header>{title}</header>
      <main>{children}</main>
      {footer && <footer>{footer}</footer>}
    </section>
  );
}

// 使用方通过组合定制
<Card title="Profile" footer={<button>Save</button>}>
  <p>Content here</p>
</Card>
```

**错误**：

```tsx
// 通过 props 枚举所有变体 —— 不可扩展，props 爆炸
interface CardProps {
  title: string;
  content: string;
  showFooter: boolean;
  footerText?: string;
  footerIcon?: string;
  footerVariant?: "primary" | "secondary";
  // 不断增长...
}
```

## 3. Hooks 使用纪律

### 3.1 遵守 Hooks 规则

Hooks 只能在函数组件或自定义 Hook 的顶层调用。禁止在条件、循环、`return` 之后调用 Hooks。始终安装 `eslint-plugin-react-hooks`，启用 `exhaustive-deps` 规则。

### 3.2 编译器优先，手动 memo 其次

**React 19 Compiler 自动 memo 组件和 Hooks。不要预先使用 `useMemo` 和 `useCallback`，直到性能分析器证明它们是瓶颈。**

**正确**：

```tsx
// React 19 编译器会为你 memo
function ExpensiveList({ items }: { items: Item[] }) {
  return items.map(item => <Item key={item.id} data={item} />);
}
```

**错误**：

```tsx
// 在 React 19 中过早的 memo 化 —— 编译器已处理
const ExpensiveList = React.memo(function ExpensiveList({ items }: { items: Item[] }) {
  return items.map(item => <Item key={item.id} data={item} />);
});
```

**例外**：传给第三方库（如图表库）的回调、或编译器无法推断的极少数复杂场景，才手动使用 `useCallback`/`useMemo`。

### 3.3 禁止用 `useEffect` 推导状态

**任何能从 props 或现有 state 计算的值，在渲染期间计算。** `useEffect` 仅用于同步外部系统（浏览器 API、WebSocket、非 React 状态）。

**正确**：

```tsx
function TodoList({ todos, filter }: { todos: Todo[]; filter: string }) {
  const filteredTodos = todos.filter(t => t.text.includes(filter));
  // 渲染期间计算 —— 无 effect
  return <ul>{filteredTodos.map(t => <li key={t.id}>{t.text}</li>)}</ul>;
}
```

**错误**：

```tsx
function TodoList({ todos, filter }: { todos: Todo[]; filter: string }) {
  const [filteredTodos, setFilteredTodos] = useState(todos);
  useEffect(() => {
    setFilteredTodos(todos.filter(t => t.text.includes(filter)));
  }, [todos, filter]);
  // 不必要地多了一次重渲染
}
```

## 4. 表单与 Actions（React 19）

**使用 `<form action={...}>` 配合 Actions 处理数据提交，自动管理 pending/error 状态。**

**正确**：

```tsx
function CreateUser() {
  const [error, formAction, isPending] = useActionState(
    async (_prev: unknown, formData: FormData) => {
      const res = await fetch("/api/users", {
        method: "POST",
        body: formData,
      });
      if (!res.ok) return "Failed to create user";
      return null;
    },
    null
  );

  return (
    <form action={formAction}>
      <input name="name" required />
      {error && <p role="alert">{error}</p>}
      <button type="submit" disabled={isPending}>
        {isPending ? "Creating..." : "Create"}
      </button>
    </form>
  );
}
```

**错误**：手动管理表单提交状态。

```tsx
// 手动管理所有状态 —— Actions 已内置这些
function CreateUser() {
  const [name, setName] = useState("");
  const [error, setError] = useState<string | null>(null);
  const [isPending, setIsPending] = useState(false);

  async function handleSubmit(e: FormEvent) {
    e.preventDefault();
    setIsPending(true);
    try {
      const res = await fetch("/api/users", { method: "POST", body: new FormData(e.target as HTMLFormElement) });
      if (!res.ok) throw new Error("Failed");
    } catch (err) {
      setError(err.message);
    } finally {
      setIsPending(false);
    }
  }
  // ...
}
```

## 5. 逻辑/视图分离模式

**将组件逻辑提取到独立的 `.ts`/`.tsx` 文件中，而非全部写在 JSX 内。**

对于复杂组件，UI 渲染和业务逻辑应分离到不同文件：

```
src/
├── components/
│   ├── Dashboard.tsx          ← 仅包含 JSX 模板
│   ├── Dashboard.hooks.ts     ← 自定义 Hooks（数据获取、事件绑定）
│   └── Dashboard.utils.ts     ← 纯工具函数
```

**正确**：

```tsx
// components/Dashboard.hooks.ts —— 逻辑层
import { useState, useEffect } from "react";
import { listen } from "@tauri-apps/api/event";

interface ControllerData {
  buttons: number;
  left_stick: { x: number; y: number };
}

export function useControllerEvents() {
  const [data, setData] = useState<ControllerData | null>(null);

  useEffect(() => {
    const unlistenPromise = listen<ControllerData>("controller_update", (event) => {
      setData(event.payload);
    });
    return () => {
      unlistenPromise.then((unlisten) => unlisten()); // 清理
    };
  }, []);

  return data;
}
```

```tsx
// components/Dashboard.tsx —— 视图层
import { useControllerEvents } from "./Dashboard.hooks";

export function Dashboard() {
  const controllerData = useControllerEvents();

  return <div>{controllerData?.buttons}</div>;
}
```

**错误**：将所有逻辑和事件绑定直接写在组件中，组件膨胀到数百行。

## 6. 状态管理

### 6.1 集中式状态管理（桌面应用推荐模式）

对于 Tauri/Electron 等桌面应用，建议使用 zustand 或 React Context 管理全局状态。这比分散的多个 Context Provider 更适合桌面应用高频实时数据更新的场景。

```typescript
// stores/app-store.ts
import { create } from "zustand";

interface AppSettings {
  autoStart: boolean;
  minimizeToTray: boolean;
  theme: string;
}

interface AppState {
  // 设备状态
  isConnected: boolean;
  deviceName: string;

  // 设置（从后端加载）
  settings: AppSettings | null;

  // UI 状态
  statusMessage: string;
  showModal: boolean;

  // Actions
  setConnected: (name: string) => void;
  setDisconnected: () => void;
  setStatus: (msg: string) => void;
}

export const useAppStore = create<AppState>((set) => ({
  isConnected: false,
  deviceName: "",
  settings: null,
  statusMessage: "Ready",
  showModal: false,

  setConnected: (name) => set({ isConnected: true, deviceName: name }),
  setDisconnected: () => set({ isConnected: false, deviceName: "" }),
  setStatus: (msg) => set({ statusMessage: msg }),
}));
```

```tsx
// 在组件中使用
import { useAppStore } from "@/stores/app-store";

function StatusBar() {
  const isConnected = useAppStore((s) => s.isConnected);
  const statusMessage = useAppStore((s) => s.statusMessage);

  return <div>{isConnected ? "Connected" : statusMessage}</div>;
}
```

**推荐原则**：
- 设备状态、应用设置、UI 状态放在 centralized store 中
- 页面级临时状态用 `useState`
- 不在 store 中存放后端数据的"镜像副本"——通过数据请求库缓存更新

### 6.2 colocate 状态到最近的使用方

**在需要的组件内部定义 state，仅在兄弟节点需要共享时才提升。**

全局状态（auth、theme、current org）优先用 Context + useReducer 或小型 store（zustand）。禁止将服务端数据存放在客户端 state 中——通过 Suspense 或数据请求库获取。

**正确**：

```tsx
function SearchBar() {
  const [query, setQuery] = useState(""); // state 在需要的组件内
  return <input value={query} onChange={e => setQuery(e.target.value)} />;
}
```

**错误**：

```tsx
// 预判性地将 state 提升到父组件"以防万一"
function Parent() {
  const [query, setQuery] = useState(""); // 此时只有一个 child 需要
  return <SearchBar query={query} onQueryChange={setQuery} />;
}
```

### 6.3 使用 `useOptimistic` 处理乐观更新

```tsx
"use client";

function LikeButton({ postId, initialLikes }: { postId: string; initialLikes: number }) {
  const [optimisticLikes, addOptimisticLike] = useOptimistic(
    initialLikes,
    (_state, _newLikes: number) => _state + 1
  );

  async function handleLike() {
    addOptimisticLike(optimisticLikes);
    await fetch(`/api/posts/${postId}/like`, { method: "POST" });
  }

  return <button onClick={handleLike}>♥ {optimisticLikes}</button>;
}
```

## 7. 类型安全

**所有组件 props 必须使用 TypeScript 接口定义。禁止使用 `propTypes`（React 19 已移除）。**

**正确**：

```tsx
interface ButtonProps {
  variant: "primary" | "secondary";
  disabled?: boolean;
  children: React.ReactNode;
}

export function Button({ variant, disabled, children }: ButtonProps) {
  // ...
}
```

泛型组件保持类型安全：

```tsx
interface ListProps<T> {
  items: T[];
  renderItem: (item: T) => React.ReactNode;
}

export function List<T>({ items, renderItem }: ListProps<T>) {
  return <ul>{items.map(renderItem)}</ul>;
}
```

## 8. 项目结构

**按功能/领域组织文件，而非按类型。**

```
src/
├── app/              # Next.js App Router 页面路由
├── components/       # 通用组件
│   ├── ui/           # 基础 UI 组件（Button, Input, Card）
│   └── shared/       # 业务共享组件
├── features/         # 按功能域划分
│   ├── auth/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── api.ts
│   │   └── types.ts
│   └── dashboard/
│       ├── components/
│       ├── hooks/
│       └── api.ts
├── lib/              # 工具函数、API 客户端
└── types/            # 全局类型定义
```

## 9. 快速参考

| 规则 | 核心要求 | 主要例外 |
|------|---------|---------|
| Server First | 默认 Server Component | 需要交互性时加 `"use client"` |
| 组合优于配置 | `children` + slot 替代 boolean props 爆炸 | 简单变体用 props |
| 禁用 `useEffect` 推导 | 渲染期间计算派生值 | 同步外部系统 |
| Actions 管理表单 | `<form action>` + `useActionState` | 无需表单状态的简单交互 |
| Compiler 替代手动 memo | 不用 `useMemo`/`useCallback` 预优化 | profiler 证明是瓶颈时 |
| colocate state | state 定义在使用方 | 兄弟节点共享时提升 |
| TypeScript props | `interface` 替代 `propTypes` | 无 |

## 10. 禁止事项

- 禁止在 Server Component 中使用 `useState`、`useEffect`、事件处理器
- 禁止使用 `useEffect` 推导派生状态
- 禁止在 React 19 中无理由地手动 `React.memo`/`useMemo`/`useCallback`
- 禁止使用 `propTypes`（React 19 已移除）
- 禁止在循环/条件/`return` 后调用 Hooks
- 禁止将服务端数据存放到客户端 `useState` 中
- 禁止在组件内部定义另一个组件（破坏稳定性）

## 11. 自检要求

输出 React 代码或配置前，逐项检查：

- [ ] Server Component 和 Client Component 的边界是否正确（`"use client"` 仅在有交互需求时添加）？
- [ ] 没有多余的 `useMemo`/`useCallback`（React 19 Compiler 已自动处理）？
- [ ] 表单提交使用 Actions（`useActionState` + `<form action>`）而非手动管理状态？
- [ ] 没有 `useEffect` 用于推导状态？
- [ ] state 创建在最近的使用方，没有预判性提升？
- [ ] 组件 props 使用 TypeScript 接口，没有 `propTypes`？
- [ ] 组合优先于配置（`children` 替代 boolean props 爆炸）？
- [ ] `eslint-plugin-react-hooks` 已安装且 `exhaustive-deps` 启用？
