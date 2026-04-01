# TypeScript 速查手册（面向 Python 开发者）

> 本文档针对 Claude Code 项目中实际使用的 TypeScript 语法编写，帮助 Python 开发者快速上手。

---

## 目录

1. [基础类型与类型注解](#1-基础类型与类型注解)
2. [接口与类型别名](#2-接口与类型别名)
3. [联合类型与交叉类型](#3-联合类型与交叉类型)
4. [可选属性与可选链](#4-可选属性与可选链)
5. [数组与元组](#5-数组与元组)
6. [泛型](#6-泛型)
7. [类型守卫与类型断言](#7-类型守卫与类型断言)
8. [函数](#8-函数)
9. [类](#9-类)
10. [模块系统](#10-模块系统)
11. [异步编程](#11-异步编程)
12. [解构与展开](#12-解构与展开)
13. [模板字符串](#13-模板字符串)
14. [枚举](#14-枚举)
15. [实用工具类型](#15-实用工具类型)
16. [高级类型操作](#16-高级类型操作)
17. [satisfies 运算符](#17-satisfies-运算符)
18. [void 运算符](#18-void-运算符)
19. [声明文件 (.d.ts)](#19-声明文件-dts)
20. [Python vs TypeScript 速查对照表](#20-python-vs-typescript-速查对照表)

---

## 1. 基础类型与类型注解

TypeScript 在 JavaScript 基础上添加了**静态类型系统**。Python 使用 `: type` 做类型注解（PEP 484），TypeScript 的语法非常相似但更严格。

```typescript
// 基本类型 —— Python 对应：str, int, float, bool, None
const name: string = "Claude"
const age: number = 30          // Python 没有 int/float 区分，TS 统一为 number
const active: boolean = true
const nothing: null = null
const empty: undefined = undefined

// any —— 相当于 Python 的不注解（任意类型），尽量少用
let data: any = "hello"
data = 42    // 不报错，但失去类型保护

// unknown —— 比 any 更安全的"未知类型"，使用前必须做类型检查
let input: unknown = fetchData()
if (typeof input === "string") {
  console.log(input.toUpperCase())  // 只有检查后才允许使用
}

// void —— 函数不返回值，类似 Python 的 -> None
function log(msg: string): void {
  console.log(msg)
}

// never —— 函数永远不会返回（抛异常或死循环）
function fail(msg: string): never {
  throw new Error(msg)
}
```

**与 Python 的关键区别：**
- TypeScript 的 `number` 不区分 `int` 和 `float`
- TypeScript 区分 `null` 和 `undefined`（Python 只有 `None`）
- TypeScript 编译时检查类型，Python 的类型注解只是提示（除非用 mypy/pyright）

---

## 2. 接口与类型别名

Python 没有原生的接口概念（通常用 `dataclass` 或 `TypedDict` 代替），TypeScript 有两种定义对象形状的方式。

### interface（接口）

```typescript
interface User {
  name: string
  age: number
  email?: string        // 可选属性，相当于 Python 的 Optional
  readonly id: number   // 只读属性，初始化后不能修改
}

const user: User = { name: "Alice", age: 30, id: 1 }
user.name = "Bob"       // OK
// user.id = 2          // 报错：id 是 readonly
```

### type（类型别名）

```typescript
type Point = {
  x: number
  y: number
}

// type 可以表示任何类型，不只是对象
type ID = string | number
type Callback = (data: string) => void
```

### interface vs type 怎么选？

| 特性 | interface | type |
|------|-----------|------|
| 对象形状 | 推荐 | 可以 |
| 联合类型 | 不行 | `type A = B \| C` |
| 交叉类型 | 用 extends | `type A = B & C` |
| 声明合并 | 支持（自动合并同名接口） | 不支持 |

**本项目中的用法** — `interface` 和 `type` 都大量使用：
```typescript
// src/QueryEngine.ts
export type QueryEngineConfig = {
  cwd: string
  tools: Tools
  commands: Command[]
  mcpClients: MCPServerConnection[]
  // ...
}
```

---

## 3. 联合类型与交叉类型

### 联合类型（Union Types）—— 类似 Python 的 `Union[A, B]`

```typescript
// 变量可以是多种类型之一
type Status = "active" | "inactive" | "pending"   // 字面量联合类型（类似 Python Literal）
type ID = string | number                          // 类型联合

function processId(id: ID) {
  // 必须先用类型守卫收窄类型
  if (typeof id === "string") {
    return id.toUpperCase()     // TS 知道这里 id 是 string
  }
  return id.toFixed(2)          // TS 知道这里 id 是 number
}
```

### 交叉类型（Intersection Types）—— 类似 Python 的多继承/Protocol 合并

```typescript
type Named = { name: string }
type Aged = { age: number }
type Person = Named & Aged       // 同时拥有 name 和 age

const p: Person = { name: "Alice", age: 30 }
```

**本项目中的用法** — 鉴别联合（Discriminated Unions）：
```typescript
// 通过 type 字段区分不同种类的消息
type TextBlock = {
  type: "text"          // 字面量类型作为"鉴别标签"
  text: string
}

type ImageBlock = {
  type: "image"
  url: string
}

type ContentBlock = TextBlock | ImageBlock

function render(block: ContentBlock) {
  switch (block.type) {    // TS 根据 type 自动收窄类型
    case "text":
      console.log(block.text)     // 自动推断为 TextBlock
      break
    case "image":
      console.log(block.url)      // 自动推断为 ImageBlock
      break
  }
}
```

---

## 4. 可选属性与可选链

### 可选属性 `?`

```typescript
interface Config {
  host: string
  port?: number           // 可选，相当于 port: number | undefined
}

const config: Config = { host: "localhost" }   // port 可以不提供
```

### 可选链 `?.` —— Python 3.10 没有，类似 `getattr(obj, 'attr', None)?.method()`

```typescript
const user = { address: { street: "Main St" } }

// 如果中间某个属性是 null/undefined，不会报错，而是返回 undefined
console.log(user?.address?.zip)           // undefined（不报错）
console.log(user?.getName?.())            // undefined（方法调用也可选链）
console.log(user?.hobbies?.[0])           // undefined（数组索引也可选链）
```

### 空值合并 `??` —— 类似 Python 的 `value or default`，但只对 null/undefined 生效

```typescript
const port = config.port ?? 3000          // port 为 null/undefined 时用 3000
const name = "" ?? "default"              // "" —— 空字符串不会被替换
const name2 = "" || "default"             // "default" —— || 会把 falsy 值都替换
```

**本项目中的用法：**
```typescript
// src/services/api/withRetry.ts
function getMaxRetries(options: RetryOptions): number {
  return options.maxRetries ?? getDefaultMaxRetries()    // 未指定则用默认值
}
```

---

## 5. 数组与元组

### 数组 —— 对应 Python 的 list

```typescript
const numbers: number[] = [1, 2, 3]
const strings: Array<string> = ["a", "b", "c"]    // 等价写法

// 数组方法（与 Python 列表推导对应）
numbers.map(n => n * 2)           // 对应 [n * 2 for n in numbers]
numbers.filter(n => n > 1)        // 对应 [n for n in numbers if n > 1]
numbers.reduce((sum, n) => sum + n, 0)  // 对应 sum(numbers)
```

### 只读数组

```typescript
const readonlyArr: readonly number[] = [1, 2, 3]
// readonlyArr.push(4)  // 报错
```

### 元组 —— 对应 Python 的 tuple

```typescript
// 固定长度、固定类型的数组
const pair: [string, number] = ["Alice", 30]
const name = pair[0]     // string
const age = pair[1]      // number

// 只读元组
const point: readonly [number, number] = [0, 0]
```

**本项目中的用法：**
```typescript
// src/hooks/useVirtualScroll.ts
range: readonly [number, number]
```

---

## 6. 泛型

泛型是 TypeScript 最强大的特性之一。Python 3.12 引入了 `class MyClass[T]:` 语法，概念类似。

### 基本用法

```typescript
// 泛型函数 —— 类似 Python 的 def first[T](items: list[T]) -> T
function first<T>(items: T[]): T | undefined {
  return items[0]
}

first([1, 2, 3])          // 返回 number
first(["a", "b"])         // 返回 string

// 泛型接口
interface Box<T> {
  value: T
}

const numberBox: Box<number> = { value: 42 }
const stringBox: Box<string> = { value: "hello" }
```

### 泛型约束 `extends`

```typescript
// T 必须有 length 属性
function logLength<T extends { length: number }>(item: T): void {
  console.log(item.length)
}

logLength("hello")       // OK，string 有 length
logLength([1, 2, 3])     // OK，array 有 length
// logLength(123)        // 报错，number 没有 length

// 多个泛型参数
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key]
}
```

### 泛型与数组方法

```typescript
// 本项目中的真实用法 —— reduce 的泛型
const cleanedCount = cleanupResults.reduce<number>((sum, n) => sum + n, 0)
//                        泛型参数 ^^^^^^ 告诉 reduce 累加器是 number 类型
```

---

## 7. 类型守卫与类型断言

### 类型守卫（Type Guards）—— 让 TS 自动收窄类型

```typescript
// typeof 守卫
function pad(value: string | number): string {
  if (typeof value === "string") {
    return value           // TS 知道这里是 string
  }
  return value.toString()  // TS 知道这里是 number
}

// instanceof 守卫
function handleError(error: unknown): string {
  if (error instanceof Error) {
    return error.message    // TS 知道这里是 Error
  }
  return String(error)
}

// "in" 操作符守卫
interface Fish { swim(): void }
interface Bird { fly(): void }
function move(animal: Fish | Bird) {
  if ("swim" in animal) {
    animal.swim()           // TS 知道是 Fish
  } else {
    animal.fly()            // TS 知道是 Bird
  }
}
```

### 类型断言 `as` —— 告诉 TS "我比你更清楚类型"

```typescript
// 类似 Python 的 cast(SomeType, value)，但运行时不起作用
const input = document.getElementById("input") as HTMLInputElement

// 本项目中的真实用法
const error = lastError as APIError
error.status    // TS 现在认为 error 有 status 属性
```

### asserts 类型守卫

```typescript
// 本项目中的真实用法 —— 如果函数正常返回，则 TS 知道 scope 一定是 InstallableScope
function assertInstallableScope(scope: string): asserts scope is InstallableScope {
  if (!VALID_INSTALLABLE_SCOPES.includes(scope as InstallableScope)) {
    throw new Error(`Invalid scope "${scope}"`)
  }
}

// 调用后 TS 自动收窄类型
assertInstallableScope(myScope)   // 此后 myScope 的类型从 string 收窄为 InstallableScope
```

---

## 8. 函数

### 函数声明

```typescript
// 基本函数
function add(a: number, b: number): number {
  return a + b
}

// 箭头函数 —— 类似 Python lambda 但更强大
const multiply = (a: number, b: number): number => a * b

// 默认参数（与 Python 相同）
function greet(name: string, greeting = "Hello"): string {
  return `${greeting}, ${name}`
}
```

### 函数重载

Python 没有重载（通常用 `*args/**kwargs` 或 `isinstance` 判断），TypeScript 支持签名重载：

```typescript
function process(input: string): number
function process(input: number): string
function process(input: string | number): string | number {
  if (typeof input === "string") {
    return input.length
  }
  return input.toString()
}
```

### 回调函数类型

```typescript
// 定义回调的形状
type Callback = (error: Error | null, result?: string) => void

function fetchData(cb: Callback) {
  // ...
}

// 本项目中的真实用法
type CanUseToolFn = (
  toolName: string,
  input: Record<string, unknown>,
) => Promise<boolean>
```

---

## 9. 类

TypeScript 的类比 Python 的类更接近 Java/C#，有访问修饰符。

```typescript
class QueryEngine {
  private messages: Message[]              // 私有，类外部不可访问
  protected config: QueryEngineConfig      // 保护，子类可访问
  public readonly id: string               // 公开只读

  constructor(config: QueryEngineConfig) {  // 构造函数（对应 __init__）
    this.config = config
    this.id = generateId()
    this.messages = []
  }

  // 方法
  async submitMessage(msg: string): Promise<void> {
    // ...
  }

  // 静态方法（对应 @staticmethod）
  static create(config: QueryEngineConfig): QueryEngine {
    return new QueryEngine(config)
  }
}
```

### 抽象类 —— 类似 Python 的 ABC

```typescript
abstract class BaseTool {
  abstract name: string
  abstract execute(input: unknown): Promise<unknown>

  describe(): string {
    return `Tool: ${this.name}`
  }
}
```

---

## 10. 模块系统

TypeScript 使用 ES Module（与 Python 的 import 机制不同）。

```typescript
// 命名导出（可以导出多个）
export interface User { name: string }
export function getUser(): User { /* ... */ }
export const DEFAULT_NAME = "Anonymous"

// 命名导入
import { User, getUser, DEFAULT_NAME } from "./user"

// 导入并重命名
import { getUser as fetchUser } from "./user"

// 全部导入
import * as userModule from "./user"
userModule.getUser()

// 默认导出（每个文件最多一个）
export default class MyApp { /* ... */ }

// 默认导入（不需要花括号）
import MyApp from "./app"

// 类型导入（仅导入类型信息，编译后被移除）
import type { User } from "./user"
```

**Python 对比：**
```python
# Python
from user import User, get_user
from user import get_user as fetch_user

# TypeScript
import { User, getUser } from "./user"
import { getUser as fetchUser } from "./user"
```

**关键区别：**
- TypeScript 导入路径必须写相对路径和扩展名（如 `"./user"`）
- `export default` 在 Python 中没有直接对应

---

## 11. 异步编程

TypeScript 的 async/await 语法与 Python 非常相似，但额外有 **Promise** 和**异步生成器**。

### Promise —— 类似 Python 的 asyncio.Future

```typescript
// Promise 是异步操作的容器
const promise: Promise<string> = fetch("https://api.example.com")
  .then(res => res.text())

// 更好的写法：async/await（与 Python 语法几乎相同）
async function fetchData(): Promise<string> {
  const response = await fetch("https://api.example.com")
  return response.text()
}
```

### async/await —— 与 Python 几乎相同

```typescript
// Python:
// async def get_data() -> str:
//     response = await fetch(url)
//     return await response.text()

async function getData(): Promise<string> {
  const response = await fetch(url)
  return response.text()
}
```

### 异步生成器（本项目大量使用）

Python 的异步生成器用 `async def` + `yield`，TypeScript 用 `async function*`：

```typescript
// 本项目中的真实模式 —— async generator
async function* queryLoop(
  params: QueryParams
): AsyncGenerator<Message> {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const result = await apiCall()
      yield result                   // 产出一条消息
    } catch (error) {
      yield createErrorMessage(error)  // 即使出错也产出消息
      continue
    }
  }
}

// yield* —— 委托给另一个生成器（Python 也有 yield from）
async function* processMessages() {
  yield* normalizeMessage(message)   // 把 normalizeMessage 的产出转发出去
}

// 消费异步生成器
for await (const message of queryLoop(params)) {
  console.log(message)
}
```

### `void` 表达式 —— "发射后不管"的异步调用

```typescript
// 本项目中的真实用法
void recordTranscript(messages)    // 不 await，让它在后台执行
void fileHistoryMakeSnapshot(cwd)  // 类似 Python 的 asyncio.create_task() 但不等待结果
```

---

## 12. 解构与展开

### 对象解构 —— 类似 Python 的多重赋值

```typescript
const { name, age } = user                // const name = user.name
const { name: userName, ...rest } = user  // 重命名 + 收集剩余属性

// 函数参数解构
function greet({ name, age }: User): string {
  return `${name}, age ${age}`
}
```

### 数组解构

```typescript
const [first, second, ...rest] = [1, 2, 3, 4]
// first = 1, second = 2, rest = [3, 4]

// 交换变量（Python: a, b = b, a）
[a, b] = [b, a]
```

### 展开运算符 `...` —— 类似 Python 的 `*args` / `**kwargs`

```typescript
// 对象展开（类似 {**obj1, **obj2}）
const merged = { ...defaults, ...userConfig }

// 数组展开（类似 [*list1, *list2]）
const combined = [...arr1, ...arr2, newItem]
```

---

## 13. 模板字符串

```typescript
// 反引号模板 —— 类似 Python f-string
const greeting = `Hello, ${name}!`
const multiLine = `
  第一行
  第二行
`

// 带标签的模板（高级用法）
function sql(strings: TemplateStringsArray, ...values: unknown[]) {
  // 可以对模板做自定义处理
}
```

---

## 14. 枚举

```typescript
// 数字枚举（默认从 0 开始）
enum Direction {
  Up,       // 0
  Down,     // 1
  Left,     // 2
  Right,    // 3
}

// 字符串枚举
type PermissionMode = "default" | "plan" | "autoAccept"

// const enum（编译时内联，不生成 JS 代码）
const enum ErrorCode {
  NotFound = 404,
  ServerError = 500,
}
```

**注意**：本项目中更常用**字面量联合类型**（`type X = "a" | "b"`）而非 `enum`，这在 TypeScript 社区也是推荐做法。

---

## 15. 实用工具类型

TypeScript 内置了许多工具类型，Python 没有直接对应。

### `Partial<T>` / `Required<T>`

```typescript
interface User {
  name: string
  age: number
  email: string
}

type PartialUser = Partial<User>
// 等价于 { name?: string; age?: number; email?: string }

type RequiredUser = Required<PartialUser>
// 所有属性变为必填
```

### `Pick<T, K>` / `Omit<T, K>`

```typescript
type UserName = Pick<User, "name" | "email">
// { name: string; email: string }

type UserWithoutEmail = Omit<User, "email">
// { name: string; age: number }
```

### `Record<K, V>`

```typescript
// 类似 Python 的 dict[str, str]
type Config = Record<string, string>
// 等价于 { [key: string]: string }
```

### `ReturnType<T>` / `Parameters<T>`

```typescript
function createUser(name: string, age: number) {
  return { name, age, id: Math.random() }
}

type User = ReturnType<typeof createUser>         // { name: string; age: number; id: number }
type CreateUserParams = Parameters<typeof createUser>  // [string, number]
```

### `Awaited<T>` —— 获取 Promise 内部的类型

```typescript
type Result = Awaited<Promise<string>>   // string
```

### `Extract<T, U>` / `Exclude<T, U>`

```typescript
type T = "a" | "b" | "c"
type AB = Extract<T, "a" | "b">    // "a" | "b" —— 提取交集
type C = Exclude<T, "a" | "b">     // "c" —— 排除交集
```

**本项目中的用法：**
```typescript
// src/ink/terminal-querier.ts
type DecrpmResponse = Extract<TerminalResponse, { type: 'decrpm' }>
type Da1Response = Extract<TerminalResponse, { type: 'da1' }>
```

---

## 16. 高级类型操作

### `keyof` —— 获取对象类型的所有键

```typescript
interface User {
  name: string
  age: number
}

type UserKeys = keyof User          // "name" | "age"

function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key]
}
```

### `typeof` —— 从值获取类型

```typescript
const config = {
  host: "localhost",
  port: 3000,
}

type Config = typeof config
// { host: string; port: number }
```

### 映射类型（Mapped Types）—— 类似字典推导

```typescript
// 本项目中的真实用法
type DeepImmutable<T> = T extends primitive
  ? T
  : { readonly [K in keyof T]: DeepImmutable<T[K]> }
//      ^^^^^^^^                ^^^^^^^^^^^^^^^^^^^^
//      所有属性变为只读         递归应用到嵌套类型
```

### 条件类型（Conditional Types）—— 类似三元表达式

```typescript
type IsString<T> = T extends string ? "yes" : "no"

type A = IsString<string>    // "yes"
type B = IsString<number>    // "no"
```

### `as const` —— 字面量类型断言

```typescript
// 普通 const 推断为 string[]
const arr1 = ["a", "b"] as const
// 类型变为 readonly ["a", "b"] —— 精确的字面量类型

const obj = { name: "Alice", age: 30 } as const
// { readonly name: "Alice"; readonly age: 30 }
```

---

## 17. satisfies 运算符

这是 TypeScript 4.9+ 新增的运算符，本项目中有使用。它**验证**表达式符合某类型，但**不改变**推断出的类型。

```typescript
// 本项目中的真实用法
const result = {
  type: "command",
  execute: async (input) => { /* ... */ }
} satisfies Command
// result 的类型保持精确的字面量类型，同时确保它符合 Command 接口

// 对比：
const a: Command = { ... }    // 类型被拓宽为 Command
const b = { ... } satisfies Command  // 类型保持精确，但确保符合 Command
```

---

## 18. void 运算符

```typescript
// void 表达式：执行异步函数但不等待结果
// 类似 Python 的 asyncio.create_task() 但完全不管结果
void someAsyncFunction()

// 本项目中的真实用法
void recordTranscript(messages)      // 记录日志，不阻塞主流程
void fileHistoryMakeSnapshot(cwd)    // 保存快照，不等待完成
```

---

## 19. 声明文件 (.d.ts)

声明文件为已有的 JavaScript 代码提供类型信息，Python 中没有对应概念。

```typescript
// 例如 global.d.ts
declare module "some-untyped-lib" {
  export function doSomething(input: string): number
}

// 扩展全局类型
declare global {
  interface Window {
    myCustomProp: string
  }
}

// 空的 export 使文件成为模块（而非脚本）
export {}
```

---

## 20. Python vs TypeScript 速查对照表

| 概念 | Python | TypeScript |
|------|--------|------------|
| 变量声明 | `x = 1` | `const x: number = 1` / `let x = 1` |
| 常量 | `CONST = 1` (约定全大写) | `const CONST = 1` (真正不可变) |
| 类型注解 | `x: int = 1` | `const x: number = 1` |
| 字符串 | `f"Hello {name}"` | `` `Hello ${name}` `` |
| None/空值 | `None` | `null` / `undefined` |
| 可选类型 | `Optional[str]` / `str \| None` | `string \| undefined` 或 `string?` |
| 列表 | `list[int]` | `number[]` 或 `Array<number>` |
| 字典 | `dict[str, int]` | `Record<string, int>` 或 `{ [key: string]: int }` |
| 元组 | `tuple[str, int]` | `[string, number]` |
| 联合类型 | `Union[str, int]` / `str \| int` | `string \| number` |
| 接口/协议 | `Protocol` / `TypedDict` | `interface` / `type` |
| 枚举 | `Enum` | `enum` / 字面量联合类型 |
| 异步函数 | `async def` | `async function` |
| 等待异步 | `await coro()` | `await promise` |
| 生成器 | `def gen(): yield x` | `function* gen(): { yield x }` |
| 异步生成器 | `async def gen(): yield x` | `async function* gen(): { yield x }` |
| 解构赋值 | `a, b = b, a` | `[a, b] = [b, a]` |
| 展开 | `*list1, *list2` | `...arr1, ...arr2` |
| 类 | `class Foo:` | `class Foo { }` |
| 访问控制 | `_private` (约定) | `private` / `protected` / `public` |
| 抽象类 | `ABC` + `@abstractmethod` | `abstract class` + `abstract method()` |
| 导入 | `from x import y` | `import { y } from "./x"` |
| 导出 | 无（模块级别公开） | `export` |
| 类型忽略 | `# type: ignore` | `// @ts-ignore` 或 `as any` |
| 泛型 | `T = TypeVar('T')` / `class Foo[T]:` | `function foo<T>()` / `class Foo<T>` |
| 静态方法 | `@staticmethod` | `static method()` |
| 属性 | `@property` | `get prop()` / `set prop()` |
| 装饰器 | `@decorator` | `@decorator`（实验性功能） |

---

## 附：阅读本项目源码的建议

1. **入口文件**：从 `src/cli/index.ts` 开始，这是程序的入口点
2. **核心流程**：`src/QueryEngine.ts` 是查询引擎的核心，理解它就理解了项目的主线
3. **类型定义**：`src/types/` 目录包含项目的核心类型定义，遇到不熟悉的类型可以来这里查
4. **工具系统**：`src/tools/` 目录包含各种工具的实现，每个工具都是一个独立模块
5. **遇到不认识的语法**：先查本文档，然后查阅 [TypeScript 官方文档](https://www.typescriptlang.org/docs/)
