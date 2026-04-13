# SushiSwap Swap + Pool 模块复刻开发计划书

> **版本**: v1.0  
> **日期**: 2026-04-12  
> **目标**: 通过逐步执行本计划书，完整复刻 SushiSwap 的 Swap (代币兑换) 和 Pool (流动性池) 模块  
> **源码参考**: https://github.com/sushi-labs/sushiswap

---

## 📋 目录

1. [项目总览与业务逻辑](#一项目总览与业务逻辑)
2. [架构设计分析](#二架构设计分析)
3. [Phase 1: 项目初始化与基础设施](#三phase-1-项目初始化与基础设施)
4. [Phase 2: 核心数据层开发](#四phase-2-核心数据层开发)
5. [Phase 3: Swap 模块开发](#五phase-3-swap-模块开发)
6. [Phase 4: Pool 模块开发](#六phase-4-pool-模块开发)
7. [Phase 5: 集成与测试](#七phase-5-集成与测试)

---

## 一、项目总览与业务逻辑

### 1.1 什么是 SushiSwap？

**SushiSwap** 是一个去中心化交易所（DEX），允许用户在区块链上直接交易代币，无需通过传统的中介机构。

#### 核心功能

| 功能 | 说明 | 类比理解 |
|------|------|----------|
| **Swap (兑换)** | 将一种代币换成另一种代币 | 像在银行柜台把人民币换成美元 |
| **Pool (流动性池)** | 用户提供两种代币，让其他人可以兑换 | 像银行的"资金池"，有人存钱供他人取用 |

### 1.2 Swap 模块业务逻辑（大白话版）

#### 场景：小明想用 100 USDC 换 ETH

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          用户操作流程                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. 小明打开 SushiSwap 网站                                                 │
│     ↓                                                                       │
│  2. 连接钱包（如 MetaMask）                                                 │
│     ↓                                                                       │
│  3. 选择"付出"代币：USDC，输入数量：100                                     │
│     ↓                                                                       │
│  4. 选择"获得"代币：ETH                                                     │
│     ↓                                                                       │
│  5. 系统查询流动性池，计算出能获得多少 ETH                                  │
│     ↓                                                                       │
│  6. 系统显示：预计获得 0.05 ETH，手续费 0.3%                                │
│     ↓                                                                       │
│  7. 小明点击"兑换"按钮                                                      │
│     ↓                                                                       │
│  8. 钱包弹出确认窗口，显示交易详情                                          │
│     ↓                                                                       │
│  9. 小明确认，交易提交到区块链                                              │
│     ↓                                                                       │
│  10. 等待区块确认（通常几秒到几分钟）                                       │
│     ↓                                                                       │
│  11. 交易完成，小明钱包里多了 ETH，少了 USDC                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 系统内部执行流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          系统内部流程                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐             │
│  │  用户界面  │───▶│  状态管理  │───▶│  路由计算  │───▶│  链上交互  │             │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘             │
│       │               │               │               │                     │
│       ▼               ▼               ▼               ▼                     │
│  输入代币数量    管理用户输入    找到最佳兑换    调用智能合约              │
│  选择代币类型    和交易状态      路径和价格      执行交易                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.3 Pool 模块业务逻辑（大白话版）

#### 场景：小红想提供流动性赚取收益

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          流动性提供流程                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. 小红决定为 USDC/ETH 交易对提供流动性                                    │
│     ↓                                                                       │
│  2. 需要同时存入两种代币（比如 1000 USDC + 0.5 ETH）                        │
│     ↓                                                                       │
│  3. 系统根据当前价格计算需要存入的比例                                      │
│     ↓                                                                       │
│  4. 小红确认存入数量                                                        │
│     ↓                                                                       │
│  5. 系统发放"流动性代币"（LP Token）作为凭证                                │
│     ↓                                                                       │
│  6. 其他用户在这个池子交易时支付手续费                                      │
│     ↓                                                                       │
│  7. 手续费按比例分配给所有流动性提供者                                      │
│     ↓                                                                       │
│  8. 小红可以随时取回自己的代币 + 赚取的手续费                               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.4 Swap 与 Pool 的关系

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          模块关系图                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                         ┌──────────────┐                                   │
│                         │   用户界面    │                                   │
│                         └──────┬───────┘                                   │
│                                │                                            │
│              ┌─────────────────┼─────────────────┐                         │
│              │                 │                 │                         │
│              ▼                 ▼                 ▼                         │
│       ┌──────────┐      ┌──────────┐      ┌──────────┐                   │
│       │   Swap   │◀────▶│   Pool   │◀────▶│  价格预言 │                   │
│       │  兑换模块 │      │  流动性池 │      │  价格查询 │                   │
│       └────┬─────┘      └────┬─────┘      └──────────┘                   │
│            │                 │                                            │
│            ▼                 ▼                                            │
│       ┌──────────────────────────┐                                       │
│       │      智能合约交互层       │                                       │
│       │  (与区块链上的合约通信)    │                                       │
│       └──────────────────────────┘                                       │
│                                                                             │
│  【关系说明】                                                                │
│  • Swap 依赖 Pool：兑换需要流动性池中有足够的代币                          │
│  • Pool 服务 Swap：流动性池存在的意义就是支持兑换交易                      │
│  • 两者共享价格数据：都需要查询代币价格和汇率                              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 二、架构设计分析

### 2.1 技术栈概览

| 层级 | 技术选型 | 用途 |
|------|----------|------|
| **前端框架** | Next.js 15 + React 19 | 构建用户界面 |
| **语言** | TypeScript | 类型安全的 JavaScript |
| **样式** | Tailwind CSS | 原子化 CSS 框架 |
| **UI 组件** | Radix UI + 自定义组件 | 可访问的 UI 组件 |
| **状态管理** | Zustand / React Context | 应用状态管理 |
| **区块链交互** | Wagmi + Viem | 连接钱包、发送交易 |
| **数据获取** | TanStack Query | 服务端状态管理 |
| **包管理** | pnpm + Turborepo | Monorepo 管理 |

### 2.2 项目目录结构

```
sushiswap/
├── apps/
│   ├── web/                    # 主应用 (Next.js)
│   │   ├── src/
│   │   │   ├── app/           # Next.js App Router
│   │   │   │   ├── (networks)/
│   │   │   │   │   ├── (evm)/
│   │   │   │   │   │   ├── [chainId]/
│   │   │   │   │   │   │   ├── (trade)/
│   │   │   │   │   │   │   │   └── swap/           # Swap 页面
│   │   │   │   │   │   │   └── pool/               # Pool 页面
│   │   │   │   │   │   │       ├── v2/             # V2 池子
│   │   │   │   │   │   │       └── v3/             # V3 池子
│   │   │   │   │   │   │
│   │   │   │   ├── lib/       # 核心逻辑库
│   │   │   │   │   ├── swap/  # Swap 核心逻辑
│   │   │   │   │   ├── pool/  # Pool 核心逻辑
│   │   │   │   │   ├── wagmi/ # 区块链交互
│   │   │   │   │   └── hooks/ # React Hooks
│   │   │   │   │
│   │   │   │   ├── providers/ # React Providers
│   │   │   │   └── types/     # TypeScript 类型
│   │   │   │
│   │   └── package.json
│   │
│   └── storybook/             # 组件文档
│
├── packages/
│   ├── ui/                    # 共享 UI 组件
│   ├── hooks/                 # 共享 Hooks
│   ├── graph-client/          # 图查询客户端
│   └── sushiswap/             # 核心 SDK
│
├── config/                    # 配置文件
├── scripts/                   # 脚本工具
└── package.json
```

### 2.3 核心模块依赖关系

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          模块依赖关系图                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                           页面层 (Pages)                             │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │   │
│  │  │  Swap Page   │  │  Pool Page   │  │  Position    │              │   │
│  │  │  兑换页面     │  │  池子页面     │  │  持仓页面     │              │   │
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘              │   │
│  └─────────┼────────────────┼────────────────┼──────────────────────┘   │
│            │                │                │                           │
│            ▼                ▼                ▼                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         组件层 (Components)                          │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │   │
│  │  │  TokenInput  │  │  PoolCard    │  │  TradeButton │              │   │
│  │  │  代币输入框   │  │  池子卡片     │  │  交易按钮     │              │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│            │                │                │                           │
│            ▼                ▼                ▼                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                          逻辑层 (Logic)                              │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │   │
│  │  │  useSwap     │  │  usePool     │  │  useBalance  │              │   │
│  │  │  兑换逻辑     │  │  池子逻辑     │  │  余额查询     │              │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│            │                │                │                           │
│            ▼                ▼                ▼                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        数据层 (Data)                                 │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │   │
│  │  │  Token API   │  │  Pool API    │  │  Price API   │              │   │
│  │  │  代币数据     │  │  池子数据     │  │  价格数据     │              │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│            │                │                │                           │
│            ▼                ▼                ▼                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      区块链层 (Blockchain)                           │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │   │
│  │  │   Router     │  │  Factory     │  │   Pair       │              │   │
│  │  │  路由合约     │  │  工厂合约     │  │  交易对合约   │              │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 三、Phase 1: 项目初始化与基础设施

> **阶段目标**: 搭建项目基础架构，配置开发环境，建立代码规范
> **预计耗时**: 1-2 天


### 3.1 步骤 1: 初始化 Monorepo 项目

#### 作用说明
建立项目的基础架构，使用 pnpm workspace + Turborepo 管理多个包，这是现代大型前端项目的标准做法。

#### 源码映射
- **参考文件**: `package.json`, `pnpm-workspace.yaml`, `turbo.json`
- **所在目录**: 项目根目录

#### 依赖关系
- **无上游依赖**（这是第一步）
- **下游依赖**: 所有后续步骤都依赖此步骤建立的项目结构

#### 👉 自然语言代码提示词

```
请创建一个完整的 Monorepo 项目初始化配置：

1. 创建 package.json 文件：
   - 项目名称设为 "sushiswap-replica"
   - 版本设为 "0.1.0"
   - 设为私有项目 (private: true)
   - 添加脚本：
     * "dev": "turbo run dev"
     * "build": "turbo run build"
     * "lint": "turbo run lint"
     * "typecheck": "turbo run typecheck"
   - 添加 devDependencies：
     * "turbo": "最新版本"
     * "typescript": "最新版本"
     * "@types/node": "最新版本"

2. 创建 pnpm-workspace.yaml 文件：
   - 定义 workspaces：
     * "apps/*" - 存放应用程序
     * "packages/*" - 存放共享包

3. 创建 turbo.json 文件：
   - 配置 pipeline：
     * "build" 任务：
       - outputs: [".next/**", "dist/**"]
       - dependsOn: ["^build"]
     * "dev" 任务：
       - cache: false
       - persistent: true
     * "lint" 和 "typecheck" 任务

4. 创建基础目录结构：
   - apps/ 目录
   - packages/ 目录
   - 在每个目录下创建 .gitkeep 文件

5. 创建 .gitignore 文件：
   - 忽略 node_modules
   - 忽略 .next
   - 忽略 dist
   - 忽略 .env.local
   - 忽略日志文件

6. 创建 README.md 文件：
   - 项目标题和简介
   - 安装说明
   - 开发命令
   - 项目结构说明
```

---

### 3.2 步骤 2: 配置代码规范工具

#### 作用说明
配置 Biome 作为代码格式化和 lint 工具，确保代码风格一致性，在团队开发中尤为重要。

#### 源码映射
- **参考文件**: `biome.json`
- **所在目录**: 项目根目录

#### 依赖关系
- **依赖**: 步骤 3.1（项目初始化）
- **被依赖**: 所有代码编写步骤

#### 👉 自然语言代码提示词

```
请创建代码规范配置文件：

1. 创建 biome.json 文件：
   - 配置格式化规则：
     * 使用 2 空格缩进
     * 使用单引号
     * 行尾使用分号
     * 最大行宽 100 字符
   - 配置 lint 规则：
     * 启用推荐规则
     * 禁用 "noConsoleLog" 规则
   - 配置导入排序：
     * 按类型分组排序
   - 配置 JavaScript 和 TypeScript 支持
   - 忽略以下目录：
     * node_modules
     * .next
     * dist
     * coverage

2. 更新 package.json：
   - 添加脚本：
     * "format": "biome format . --write"
     * "lint": "biome lint ."
     * "check": "biome check ."
   - 添加 devDependencies：
     * "@biomejs/biome": "最新版本"

3. 创建 .vscode/settings.json（如果使用 VSCode）：
   - 配置保存时自动格式化
   - 设置默认格式化工具为 Biome
```

---

### 3.3 步骤 3: 创建共享 UI 包

#### 作用说明
建立共享 UI 组件包，供所有应用使用，确保 UI 风格一致性，避免重复开发。

#### 源码映射
- **参考目录**: `packages/ui/`
- **参考文件**: `packages/ui/package.json`, `packages/ui/src/index.ts`

#### 依赖关系
- **依赖**: 步骤 3.1, 3.2
- **被依赖**: 所有使用 UI 组件的页面

#### 👉 自然语言代码提示词

```
请创建共享 UI 组件包：

1. 创建 packages/ui/package.json：
   - 包名设为 "@sushiswap/ui"
   - 版本 "0.1.0"
   - 主入口 "src/index.ts"
   - 类型入口 "src/index.ts"
   - 依赖：
     * "react": "^19.0.0"
     * "react-dom": "^19.0.0"
     * "tailwindcss": "^4.0.0"
     * "class-variance-authority": "最新版本"
     * "clsx": "最新版本"
     * "tailwind-merge": "最新版本"
   - devDependencies：
     * "typescript": "最新版本"
     * "@types/react": "最新版本"

2. 创建 packages/ui/tsconfig.json：
   - 配置 TypeScript 编译选项
   - 输出目录 "dist"
   - 启用 JSX

3. 创建 packages/ui/src/index.ts：
   - 导出所有组件

4. 创建工具函数 packages/ui/src/utils/cn.ts：
   - 合并 clsx 和 tailwind-merge
   - 用于条件性合并 className

5. 创建基础组件 packages/ui/src/components/button.tsx：
   - 使用 class-variance-authority 定义变体
   - 变体包括：
     * variant: default, primary, secondary, ghost, danger
     * size: sm, md, lg
   - 支持所有原生 button 属性
   - 支持 ref 转发

6. 创建基础组件 packages/ui/src/components/input.tsx：
   - 统一的输入框样式
   - 支持所有原生 input 属性
   - 支持 ref 转发

7. 创建基础组件 packages/ui/src/components/card.tsx：
   - 卡片容器组件
   - 包含 Card, CardHeader, CardContent, CardFooter 子组件

8. 创建基础组件 packages/ui/src/components/dialog.tsx：
   - 基于 Radix UI Dialog
   - 包含 Dialog, DialogTrigger, DialogContent, DialogHeader, DialogTitle, DialogDescription 组件
```

---

### 3.4 步骤 4: 初始化 Next.js 主应用

#### 作用说明
创建主应用，这是用户直接访问的 Web 应用程序，包含 Swap 和 Pool 的所有页面。

#### 源码映射
- **参考目录**: `apps/web/`
- **参考文件**: `apps/web/package.json`, `apps/web/next.config.mjs`

#### 依赖关系
- **依赖**: 步骤 3.1, 3.2, 3.3
- **被依赖**: 所有页面和组件开发

#### 👉 自然语言代码提示词

```
请创建 Next.js 主应用：

1. 创建 apps/web/package.json：
   - 包名 "@sushiswap/web"
   - 版本 "0.1.0"
   - 脚本：
     * "dev": "next dev"
     * "build": "next build"
     * "start": "next start"
     * "lint": "next lint"
     * "typecheck": "tsc --noEmit"
   - 依赖：
     * "next": "^15.0.0"
     * "react": "^19.0.0"
     * "react-dom": "^19.0.0"
     * "@sushiswap/ui": "workspace:*"
     * "wagmi": "最新版本"
     * "viem": "最新版本"
     * "@tanstack/react-query": "最新版本"
     * "zustand": "最新版本"
   - devDependencies：
     * "typescript": "最新版本"
     * "@types/react": "最新版本"
     * "@types/node": "最新版本"
     * "tailwindcss": "^4.0.0"
     * "postcss": "最新版本"

2. 创建 apps/web/next.config.mjs：
   - 配置 transpilePackages: ["@sushiswap/ui"]
   - 配置图片域名（用于代币图标）
   - 配置重定向规则

3. 创建 apps/web/tsconfig.json：
   - 配置 TypeScript
   - 设置路径别名：
     * "@/*": ["./src/*"]
     * "@sushiswap/ui": ["../../packages/ui/src"]

4. 创建 apps/web/tailwind.config.js：
   - 配置 content 路径
   - 扩展主题颜色（添加品牌色）
   - 配置插件

5. 创建 apps/web/postcss.config.cjs：
   - 配置 Tailwind CSS 插件

6. 创建 apps/web/src/app/layout.tsx：
   - 根布局组件
   - 引入全局样式
   - 配置字体
   - 添加 Providers（后续步骤实现）

7. 创建 apps/web/src/app/page.tsx：
   - 首页组件
   - 显示欢迎信息和导航链接

8. 创建 apps/web/src/app/globals.css：
   - 引入 Tailwind 指令
   - 定义 CSS 变量
   - 添加全局样式

9. 创建 apps/web/.env.example：
   - 定义环境变量模板：
     * NEXT_PUBLIC_CHAIN_ID
     * NEXT_PUBLIC_RPC_URL
     * NEXT_PUBLIC_SUBGRAPH_URL
```

---

### 3.5 步骤 5: 配置区块链连接基础设施

#### 作用说明
配置 Wagmi 和 Viem，建立与区块链的连接能力，这是 DApp 的核心基础设施。

#### 源码映射
- **参考目录**: `apps/web/src/lib/wagmi/`, `apps/web/src/providers/`
- **参考文件**: `apps/web/src/providers/providers.tsx`

#### 依赖关系
- **依赖**: 步骤 3.4
- **被依赖**: 所有需要区块链交互的功能

#### 👉 自然语言代码提示词

```
请配置区块链连接基础设施：

1. 创建 apps/web/src/lib/wagmi/config.ts：
   - 导入必要的函数和类型
   - 配置支持的链：
     * Ethereum 主网
     * Polygon
     * Arbitrum
     * Optimism
     * Base
   - 配置 RPC 端点
   - 配置钱包连接器：
     * MetaMask
     * WalletConnect
     * Coinbase Wallet
     * Injected 钱包
   - 创建 Wagmi 配置对象
   - 导出配置

2. 创建 apps/web/src/providers/wagmi-provider.tsx：
   - 创建 WagmiProvider 组件
   - 使用 WagmiConfig 包裹子组件
   - 传入配置

3. 创建 apps/web/src/providers/query-provider.tsx：
   - 创建 QueryClient 实例
   - 配置缓存选项
   - 创建 QueryClientProvider 组件

4. 创建 apps/web/src/providers/providers.tsx：
   - 组合所有 Provider
   - 按正确顺序嵌套：
     * QueryClientProvider
     * WagmiProvider
     * 其他自定义 Provider

5. 更新 apps/web/src/app/layout.tsx：
   - 引入 Providers 组件
   - 包裹整个应用

6. 创建 apps/web/src/lib/constants.ts：
   - 定义链 ID 常量
   - 定义合约地址常量
   - 定义代币地址常量
   - 定义费用常量

7. 创建 apps/web/src/lib/network.ts：
   - 定义网络配置
   - 提供网络切换功能
   - 定义网络图标和名称映射
```

---

## 四、Phase 2: 核心数据层开发

> **阶段目标**: 建立数据获取和管理层，包括代币数据、价格数据、池子数据
> **预计耗时**: 2-3 天

### 4.1 步骤 6: 创建代币数据类型定义

#### 作用说明
定义代币相关的 TypeScript 类型，这是整个应用的数据基础，确保类型安全。

#### 源码映射
- **参考目录**: `apps/web/src/types/`
- **参考文件**: 类型定义文件

#### 依赖关系
- **依赖**: 步骤 3.1-3.5
- **被依赖**: 所有使用代币数据的功能

#### 👉 自然语言代码提示词

```
请创建代币数据类型定义：

1. 创建 apps/web/src/types/token.ts：
   - 定义 Token 接口：
     * address: string (代币合约地址)
     * symbol: string (代币符号，如 "USDC")
     * name: string (代币全称)
     * decimals: number (小数位数)
     * chainId: number (所属链 ID)
     * logoURI?: string (图标 URL，可选)

   - 定义 TokenAmount 接口：
     * token: Token
     * amount: string (原始金额字符串)
     * formatted: string (格式化后的金额)

   - 定义 Currency 类型（包含原生代币）：
     * 类似 Token 但 address 为 null（表示原生代币如 ETH）

   - 定义 TokenPair 接口：
     * token0: Token
     * token1: Token

   - 定义 TokenList 接口：
     * name: string
     * tokens: Token[]

   - 创建类型守卫函数：
     * isToken(value: unknown): value is Token
     * isCurrency(value: unknown): value is Currency

2. 创建 apps/web/src/types/index.ts：
   - 导出所有类型定义

3. 创建 apps/web/src/lib/currency-from-chain-id.ts：
   - 根据链 ID 获取原生代币信息
   - 返回 Currency 对象
   - 支持 Ethereum, Polygon, Arbitrum 等链
```

---

### 4.2 步骤 7: 实现代币列表数据获取

#### 作用说明
实现获取代币列表的功能，包括从 API 获取和本地缓存，这是 Swap 功能的基础。

#### 源码映射
- **参考目录**: `apps/web/src/lib/`
- **参考文件**: 代币相关数据获取函数
21
#### 依赖关系
- **依赖**: 步骤 4.1
- **被依赖**: Swap 和 Pool 模块

#### 👉 自然语言代码提示词

```
请实现代币列表数据获取功能：

1. 创建 apps/web/src/lib/tokens/token-list.ts：
   - 定义默认代币列表 URL
   - 创建获取代币列表的函数：
     * fetchTokenList(url: string): Promise<Token[]>
     * 使用 fetch API
     * 验证返回数据格式
     * 错误处理

   - 创建获取默认代币列表的函数：
     * getDefaultTokenList(chainId: number): Promise<Token[]>
     * 根据链 ID 返回对应列表

   - 创建搜索代币的函数：
     * searchTokens(query: string, tokens: Token[]): Token[]
     * 支持按 symbol 和 name 搜索
     * 不区分大小写

2. 创建 apps/web/src/lib/tokens/token-cache.ts：
   - 使用 localStorage 缓存代币列表
   - 创建缓存键生成函数
   - 创建缓存读取函数
   - 创建缓存写入函数
   - 设置缓存过期时间（24小时）

3. 创建 apps/web/src/lib/hooks/useTokens.ts：
   - 使用 TanStack Query 的 useQuery
   - 获取代币列表
   - 支持缓存和刷新
   - 返回：
     * tokens: Token[]
     * isLoading: boolean
     * error: Error | null
     * refetch: () => void

4. 创建 apps/web/src/lib/hooks/useToken.ts：
   - 根据地址获取单个代币信息
   - 参数：address: string, chainId: number
   - 返回 Token 对象或 null

5. 创建 apps/web/src/lib/tokens/popular-tokens.ts：
   - 定义热门代币列表
   - 按链 ID 分组
   - 包含 USDC, USDT, WETH, WBTC 等常见代币
```

---

### 4.3 步骤 8: 实现价格数据获取

#### 作用说明
实现代币价格查询功能，用于显示兑换金额和计算汇率。

#### 源码映射
- **参考文件**: `apps/web/src/lib/get-all-prices.ts`

#### 依赖关系
- **依赖**: 步骤 4.1, 4.2
- **被依赖**: Swap 金额计算、池子 APR 计算

#### 👉 自然语言代码提示词

```
请实现价格数据获取功能：

1. 创建 apps/web/src/lib/prices/price-api.ts：
   - 定义价格 API 基础 URL
   - 创建获取代币价格的函数：
     * fetchTokenPrice(tokenAddress: string, chainId: number): Promise<number>
     * 返回以 USD 计价的价格

   - 创建批量获取价格的函数：
     * fetchTokenPrices(addresses: string[], chainId: number): Promise<Record<string, number>>
     * 使用单个请求获取多个代币价格

   - 创建获取汇率的函数：
     * fetchExchangeRate(tokenIn: Token, tokenOut: Token): Promise<number>
     * 计算 tokenIn 到 tokenOut 的汇率

2. 创建 apps/web/src/lib/hooks/useTokenPrice.ts：
   - 使用 useQuery 获取单个代币价格
   - 参数：tokenAddress: string, chainId: number
   - 返回：
     * price: number | undefined
     * isLoading: boolean
     * error: Error | null

3. 创建 apps/web/src/lib/hooks/useTokenPrices.ts：
   - 批量获取多个代币价格
   - 参数：addresses: string[], chainId: number
   - 返回价格映射对象

4. 创建 apps/web/src/lib/prices/price-utils.ts：
   - 创建价格格式化函数：
     * formatPrice(price: number): string
     * 小于 0.01 显示 "< $0.01"
     * 大于 1000 使用 K/M/B 缩写

   - 创建价格变化计算函数：
     * calculatePriceChange(current: number, previous: number): number
     * 返回百分比变化

5. 创建 apps/web/src/lib/prices/price-cache.ts：
   - 实现价格数据缓存
   - 缓存过期时间：5分钟
   - 使用内存缓存（价格变化频繁）
```

---

### 4.4 步骤 9: 实现余额查询功能

#### 作用说明
实现用户代币余额查询，这是 Swap 功能必需的（需要知道用户有多少代币可兑换）。

#### 源码映射
- **参考目录**: `apps/web/src/lib/wagmi/`
- **参考文件**: 余额相关 hooks

#### 依赖关系
- **依赖**: 步骤 3.5, 4.1
- **被依赖**: Swap 输入框、Pool 存入功能

#### 👉 自然语言代码提示词

```
请实现余额查询功能：

1. 创建 apps/web/src/lib/hooks/useBalance.ts：
   - 使用 Wagmi 的 useBalance hook
   - 封装为更易用的形式
   - 参数：
     * token?: Token (不传则查询原生代币余额)
     * address?: Address (不传则使用当前连接的钱包地址)
   - 返回：
     * balance: TokenAmount | undefined
     * isLoading: boolean
     * error: Error | null
     * refetch: () => void

2. 创建 apps/web/src/lib/hooks/useBalances.ts：
   - 批量查询多个代币余额
   - 参数：
     * tokens: Token[]
     * address?: Address
   - 返回：
     * balances: Record<string, TokenAmount>
     * isLoading: boolean
     * error: Error | null

3. 创建 apps/web/src/lib/hooks/useNativeBalance.ts：
   - 专门查询原生代币余额（ETH, MATIC 等）
   - 自动识别当前链的原生代币
   - 返回格式化的余额信息

4. 创建 apps/web/src/lib/balance/balance-formatter.ts：
   - 创建余额格式化函数：
     * formatBalance(balance: bigint, decimals: number): string
     * 处理大数显示
     * 保留适当小数位

   - 创建余额解析函数：
     * parseBalance(value: string, decimals: number): bigint
     * 将用户输入转换为链上格式

5. 创建 apps/web/src/lib/balance/balance-utils.ts：
   - 检查余额是否足够：
     * hasEnoughBalance(balance: TokenAmount, required: string): boolean

   - 计算最大可兑换数量（考虑 gas 费）：
     * getMaxSwapAmount(balance: TokenAmount, isNative: boolean): string
```

---

### 4.5 步骤 10: 创建全局状态管理

#### 作用说明
建立全局状态管理，用于存储用户设置、交易状态等跨组件共享的数据。

#### 源码映射
- **参考目录**: `apps/web/src/lib/store/`
- **参考文件**: Zustand store 定义

#### 依赖关系
- **依赖**: 步骤 4.1
- **被依赖**: 所有需要共享状态的组件

#### 👉 自然语言代码提示词

```
请创建全局状态管理：

1. 创建 apps/web/src/lib/store/settings-store.ts：
   - 使用 Zustand 创建设置 store
   - 状态：
     * slippageTolerance: number (滑点容忍度，默认 0.5%)
     * deadline: number (交易截止时间，默认 20 分钟)
     * maxHops: number (最大路由跳数，默认 3)
     * preferredRouter: string (首选路由)

   - 操作方法：
     * setSlippageTolerance(tolerance: number)
     * setDeadline(minutes: number)
     * setMaxHops(hops: number)
     * resetToDefaults()

   - 持久化到 localStorage

2. 创建 apps/web/src/lib/store/swap-store.ts：
   - Swap 相关状态：
     * tokenIn: Token | null
     * tokenOut: Token | null
     * amountIn: string
     * amountOut: string
     * route: Route | null
     * isCalculating: boolean
     * error: string | null

   - 操作方法：
     * setTokenIn(token: Token | null)
     * setTokenOut(token: Token | null)
     * setAmountIn(amount: string)
     * setAmountOut(amount: string)
     * switchTokens()
     * resetSwap()

3. 创建 apps/web/src/lib/store/pool-store.ts：
   - Pool 相关状态：
     * token0: Token | null
     * token1: Token | null
     * amount0: string
     * amount1: string
     * selectedPool: Pool | null
     * poolType: 'v2' | 'v3'

   - 操作方法类似 swap-store

4. 创建 apps/web/src/lib/store/index.ts：
   - 导出所有 store
   - 创建组合 hook 方便使用

5. 创建 apps/web/src/hooks/useSettings.ts：
   - 封装 settings store
   - 提供便捷访问方式
```

---

## 五、Phase 3: Swap 模块开发

> **阶段目标**: 完整实现代币兑换功能，包括路由计算、交易构建、UI 组件
> **预计耗时**: 4-5 天

### 5.1 步骤 11: 创建 Swap 路由计算逻辑

#### 作用说明
实现路由计算功能，找到最佳的兑换路径（通过哪些池子兑换能获得最多目标代币）。

#### 源码映射
- **参考目录**: `apps/web/src/lib/swap/`
- **参考文件**: 路由相关逻辑

#### 依赖关系
- **依赖**: Phase 2 所有步骤
- **被依赖**: Swap UI 组件、交易执行

#### 👉 自然语言代码提示词

```
请创建 Swap 路由计算逻辑：

1. 创建 apps/web/src/lib/swap/types.ts：
   - 定义 Route 接口：
     * path: Token[] (兑换路径上的代币)
     * pairs: Pair[] (经过的池子)
     * inputAmount: TokenAmount
     * outputAmount: TokenAmount
     * priceImpact: number (价格影响百分比)
     * gasEstimate: bigint (预估 gas)

   - 定义 SwapQuote 接口：
     * route: Route
     * executionPrice: number (执行价格)
     * minimumReceived: TokenAmount (考虑滑点后的最小获得量)
     * feeAmount: TokenAmount (手续费)

2. 创建 apps/web/src/lib/swap/routing/router.ts：
   - 创建路由计算函数：
     * findBestRoute(
         tokenIn: Token,
         tokenOut: Token,
         amountIn: string,
         options: RouteOptions
       ): Promise<Route | null>

   - 实现路径查找算法：
     * 直接路径（tokenIn -> tokenOut）
     * 双跳路径（tokenIn -> 中间代币 -> tokenOut）
     * 三跳路径（最多支持 3 跳）

   - 计算每条路径的输出金额
   - 选择输出最多的路径

3. 创建 apps/web/src/lib/swap/routing/price-impact.ts：
   - 计算价格影响：
     * calculatePriceImpact(route: Route): number
     * 返回百分比值

   - 价格影响警告级别：
     * getPriceImpactSeverity(impact: number): 'low' | 'medium' | 'high'
     * < 1%: low
     * 1-3%: medium
     * > 3%: high

4. 创建 apps/web/src/lib/swap/routing/gas-estimator.ts：
   - 估算交易 gas：
     * estimateGas(route: Route): Promise<bigint>

   - 计算 gas 费用（以 USD 计）：
     * calculateGasCost(gas: bigint, gasPrice: bigint): number

5. 创建 apps/web/src/lib/hooks/useSwapQuote.ts：
   - 使用 useQuery 获取兑换报价
   - 参数：
     * tokenIn: Token
     * tokenOut: Token
     * amountIn: string
     * slippage: number

   - 返回：
     * quote: SwapQuote | undefined
     * isLoading: boolean
     * error: Error | null
     * refetch: () => void

6. 创建 apps/web/src/lib/swap/api-base-url.ts：
   - 定义路由 API 基础 URL
   - 支持不同环境（开发/生产）
   - 配置 API 密钥
```

---

### 5.2 步骤 12: 实现交易构建和签名

#### 作用说明
实现交易构建功能，将路由转换为可执行的区块链交易。

#### 源码映射
- **参考目录**: `apps/web/src/lib/swap/`
- **参考文件**: 交易构建相关

#### 依赖关系
- **依赖**: 步骤 5.1
- **被依赖**: 交易执行按钮

#### 👉 自然语言代码提示词

```
请实现交易构建和签名功能：

1. 创建 apps/web/src/lib/swap/transaction/builder.ts：
   - 创建交易构建函数：
     * buildSwapTransaction(
         quote: SwapQuote,
         recipient: Address,
         deadline: number
       ): Promise<TransactionRequest>

   - 构建交易数据：
     * 编码函数调用
     * 设置 value（如果是 ETH 输入）
     * 设置 gasLimit
     * 设置 from 地址

2. 创建 apps/web/src/lib/swap/transaction/approver.ts：
   - 检查代币授权：
     * checkTokenAllowance(
         token: Token,
         owner: Address,
         spender: Address,
         amount: string
       ): Promise<boolean>

   - 构建授权交易：
     * buildApproveTransaction(
         token: Token,
         spender: Address,
         amount: string
       ): TransactionRequest

   - 使用无限授权选项（可选）

3. 创建 apps/web/src/lib/swap/transaction/sender.ts：
   - 发送交易：
     * sendTransaction(tx: TransactionRequest): Promise<Hash>

   - 等待交易确认：
     * waitForTransaction(hash: Hash): Promise<TransactionReceipt>

   - 处理交易错误：
     * 解析错误信息
     * 提供用户友好的错误提示

4. 创建 apps/web/src/lib/hooks/useSwap.ts：
   - 封装完整的兑换流程：
     * approve: () => Promise<void>
     * swap: () => Promise<Hash>
     * isApproving: boolean
     * isSwapping: boolean
     * needsApproval: boolean
     * txHash: Hash | null

5. 创建 apps/web/src/lib/swap/fee.ts：
   - 计算交易手续费：
     * calculateSwapFee(amount: TokenAmount): TokenAmount

   - 手续费配置：
     * 默认 0.3%
     * 可配置
```

---

### 5.3 步骤 13: 创建 Swap 输入组件

#### 作用说明
创建代币输入组件，用户选择代币和输入数量。

#### 源码映射
- **参考目录**: `apps/web/src/app/(networks)/(evm)/[chainId]/(trade)/swap/_ui/`
- **参考文件**: `simple-swap-token0-input.tsx`, `simple-swap-token1-input.tsx`

#### 依赖关系
- **依赖**: 步骤 4.2, 4.4
- **被依赖**: Swap 主组件

#### 👉 自然语言代码提示词

```
请创建 Swap 输入组件：

1. 创建 apps/web/src/components/swap/TokenInput.tsx：
   - 组件属性：
     * token: Token | null
     * amount: string
     * onTokenSelect: (token: Token) => void
     * onAmountChange: (amount: string) => void
     * balance?: TokenAmount
     * isLoading?: boolean
     * disabled?: boolean
     * label: string ("付出" 或 "获得")

   - 组件结构：
     * 标签文本
     * 输入框（数字输入）
     * 代币选择按钮（显示当前代币或"选择代币"）
     * 余额显示（如果有）
     * "最大"按钮（点击填入全部余额）

   - 样式要求：
     * 圆角边框
     * 聚焦状态样式
     * 暗色/亮色主题支持

2. 创建 apps/web/src/components/swap/TokenSelector.tsx：
   - 组件属性：
     * isOpen: boolean
     * onClose: () => void
     * onSelect: (token: Token) => void
     * chainId: number

   - 组件功能：
     * 搜索框（按 symbol/name 搜索）
     * 热门代币快捷选择
     * 代币列表（带图标、符号、余额）
     * 按余额排序
     * 虚拟列表优化（大量代币时）

   - 使用 Dialog 组件作为容器

3. 创建 apps/web/src/components/swap/SwitchTokensButton.tsx：
   - 切换两个代币位置的按钮
   - 带旋转动画效果
   - 属性：onSwitch: () => void

4. 创建 apps/web/src/components/swap/MaxButton.tsx：
   - "最大"按钮组件
   - 点击后填入全部余额
   - 考虑 gas 预留（如果是原生代币）

5. 创建 apps/web/src/components/swap/BalanceDisplay.tsx：
   - 显示代币余额
   - 格式："余额: 1,234.56 USDC"
   - 加载状态显示
```

---

### 5.4 步骤 14: 创建 Swap 信息展示组件

#### 作用说明
创建显示兑换信息（汇率、价格影响、手续费等）的组件。

#### 源码映射
- **参考目录**: `apps/web/src/app/(networks)/(evm)/[chainId]/(trade)/swap/_ui/`
- **参考文件**: `simple-swap-trade-stats.tsx`, `simple-swap-token-rate.tsx`

#### 依赖关系
- **依赖**: 步骤 5.1
- **被依赖**: Swap 主组件

#### 👉 自然语言代码提示词

```
请创建 Swap 信息展示组件：

1. 创建 apps/web/src/components/swap/TradeStats.tsx：
   - 组件属性：
     * quote: SwapQuote
     * isLoading: boolean

   - 显示内容：
     * 汇率（1 USDC = 0.0005 ETH）
     * 价格影响（带颜色标识：绿/黄/红）
     * 最低获得量（考虑滑点）
     * 手续费
     * 预估 gas 费

   - 样式：
     * 可折叠/展开
     * 信息图标带 tooltip 说明

2. 创建 apps/web/src/components/swap/ExchangeRate.tsx：
   - 显示当前汇率
   - 支持切换显示方式：
     * 1 A = ? B
     * 1 B = ? A

   - 点击切换方向
   - 显示价格变化指示器

3. 创建 apps/web/src/components/swap/PriceImpact.tsx：
   - 显示价格影响百分比
   - 根据严重程度显示不同颜色：
     * 绿色：< 1%
     * 黄色：1-3%
     * 红色：> 3%

   - 警告提示（高价格影响时）

4. 创建 apps/web/src/components/swap/MinimumReceived.tsx：
   - 显示考虑滑点后的最小获得量
   - 格式："最少获得: 0.049 ETH"
   - 说明 tooltip

5. 创建 apps/web/src/components/swap/GasEstimate.tsx：
   - 显示预估 gas 费用
   - 格式："~$5.23"
   - 显示 gas 价格来源

6. 创建 apps/web/src/components/swap/RoutePath.tsx：
   - 可视化显示兑换路径
   * 例如：USDC → WETH → ETH
   * 使用箭头连接
   * 显示经过的池子
```

---

### 5.5 步骤 15: 创建 Swap 交易按钮和确认对话框

#### 作用说明
创建交易执行按钮和确认对话框，完成用户交互流程。

#### 源码映射
- **参考目录**: `apps/web/src/app/(networks)/(evm)/[chainId]/(trade)/swap/_ui/`
- **参考文件**: `simple-swap-trade-button.tsx`, `simple-swap-trade-review-dialog.tsx`

#### 依赖关系
- **依赖**: 步骤 5.2, 5.3, 5.4
- **被依赖**: Swap 主页面

#### 👉 自然语言代码提示词

```
请创建 Swap 交易按钮和确认对话框：

1. 创建 apps/web/src/components/swap/SwapButton.tsx：
   - 组件属性：
     * tokenIn: Token | null
     * tokenOut: Token | null
     * amountIn: string
     * quote: SwapQuote | undefined
     * isLoading: boolean
     * needsApproval: boolean
     * isApproving: boolean
     * isSwapping: boolean
     * onApprove: () => void
     * onSwap: () => void

   - 按钮状态逻辑：
     * 未选择代币 → "选择代币"（禁用）
     * 未输入金额 → "输入金额"（禁用）
     * 计算中 → "计算中..."（禁用）
     * 需要授权 → "授权 USDC"
     * 授权中 → "授权中..."
     * 可以兑换 → "兑换"
     * 兑换中 → "确认中..."
     * 余额不足 → "余额不足"（禁用）
     * 价格影响过高 → "仍要兑换"（警告样式）

2. 创建 apps/web/src/components/swap/TradeReviewDialog.tsx：
   - 组件属性：
     * isOpen: boolean
     * onClose: () => void
     * onConfirm: () => void
     * quote: SwapQuote
     * tokenIn: Token
     * tokenOut: Token
     * amountIn: string

   - 对话框内容：
     * 标题："确认兑换"
     * 付出：代币图标 + 数量 + 符号
     * 向下箭头图标
     * 获得：代币图标 + 数量 + 符号
     * 分隔线
     * 汇率信息
     * 价格影响
     * 最低获得量
     * 手续费详情
     * 确认按钮
     * 取消按钮

   - 动画效果：
     * 打开/关闭动画
     * 内容淡入

3. 创建 apps/web/src/components/swap/ApprovalDialog.tsx：
   - 授权确认对话框
   - 说明授权用途
   - 显示授权金额
   - 无限授权选项

4. 创建 apps/web/src/components/swap/TransactionPending.tsx：
   - 交易等待状态显示
   - 加载动画
   * 显示交易哈希
   * 区块链浏览器链接
   * 预估等待时间

5. 创建 apps/web/src/components/swap/TransactionSuccess.tsx：
   - 交易成功提示
   - 成功动画/图标
   - 交易摘要
   * 添加代币到钱包按钮
   * 查看交易链接
   * 关闭按钮

6. 创建 apps/web/src/components/swap/TransactionError.tsx：
   - 交易失败提示
   - 错误图标
   - 错误信息
   * 重试按钮
   * 关闭按钮
```

---

### 5.6 步骤 16: 创建 Swap 设置组件

#### 作用说明
创建兑换设置界面，允许用户调整滑点容忍度、交易截止时间等参数。

#### 源码映射
- **参考目录**: `apps/web/src/app/(networks)/(evm)/[chainId]/(trade)/swap/_ui/`
- **参考文件**: `simple-swap-settings-overlay.tsx`

#### 依赖关系
- **依赖**: 步骤 4.5
- **被依赖**: Swap 主组件

#### 👉 自然语言代码提示词

```
请创建 Swap 设置组件：

1. 创建 apps/web/src/components/swap/SettingsButton.tsx：
   - 设置按钮（齿轮图标）
   - 点击打开设置面板
   - 显示当前滑点设置

2. 创建 apps/web/src/components/swap/SettingsPanel.tsx：
   - 组件属性：
     * isOpen: boolean
     * onClose: () => void

   - 设置选项：
     * 滑点容忍度：
       - 预设按钮：0.1%, 0.5%, 1%
       - 自定义输入框
       - 警告（设置过高时）

     * 交易截止时间：
       - 预设：10, 20, 30 分钟
       - 自定义输入

     * 专家模式开关（可选）
     * 禁用多路由开关（可选）

   - 使用 Popover 或 Sheet 组件

3. 创建 apps/web/src/components/swap/SlippageInput.tsx：
   - 滑点输入组件
   - 数字输入框
   - 百分比后缀
   - 验证输入范围（0-50%）
   - 警告提示（> 5% 时）

4. 创建 apps/web/src/components/swap/DeadlineInput.tsx：
   - 截止时间输入
   - 数字输入框
   - 分钟单位
   - 验证最小值

5. 创建 apps/web/src/components/swap/ToggleOption.tsx：
   - 开关选项组件
   - 标签 + 开关
   - 说明文字
```

---

### 5.7 步骤 17: 组装 Swap 主页面

#### 作用说明
将所有 Swap 组件组装成完整的页面，实现完整的用户流程。

#### 源码映射
- **参考目录**: `apps/web/src/app/(networks)/(evm)/[chainId]/(trade)/swap/`
- **参考文件**: `page.tsx`, `simple-swap-widget.tsx`

#### 依赖关系
- **依赖**: 步骤 5.3, 5.4, 5.5, 5.6
- **被依赖**: 无（这是最终页面）

#### 👉 自然语言代码提示词

```
请组装 Swap 主页面：

1. 创建 apps/web/src/components/swap/SwapWidget.tsx：
   - 主要的兑换组件
   - 包含所有子组件的组合

   - 状态管理：
     * tokenIn, tokenOut
     * amountIn, amountOut
     * quote
     * isReviewOpen
     * txStatus: 'idle' | 'pending' | 'success' | 'error'

   - 处理函数：
     * handleTokenInSelect
     * handleTokenOutSelect
     * handleAmountInChange
     * handleSwitchTokens
     * handleSwap
     * handleApprove

   - 布局结构：
     * 卡片容器
     * 头部（标题 + 设置按钮）
     * TokenInput（付出）
     * 切换按钮
     * TokenInput（获得）
     * 兑换信息（TradeStats）
     * 兑换按钮
     * 确认对话框
     * 交易状态提示

2. 创建 apps/web/src/app/[chainId]/swap/page.tsx：
   - Next.js 页面组件
   - 获取链 ID 参数
   - 验证链是否支持
   - 渲染 SwapWidget
   - 添加页面元数据

3. 创建 apps/web/src/app/[chainId]/swap/layout.tsx：
   - 页面布局
   - 添加头部导航
   - 设置页面标题

4. 创建 apps/web/src/components/swap/SwapHeader.tsx：
   - 页面头部
   - 标题："兑换"
   - 副标题说明
   - 返回按钮

5. 创建 apps/web/src/components/swap/SwapContainer.tsx：
   - 页面容器
   - 居中布局
   * 最大宽度限制
   * 响应式内边距

6. 添加错误边界：
   - 创建 apps/web/src/app/[chainId]/swap/error.tsx
   - 显示友好的错误信息
   - 重试按钮
```

---

## 六、Phase 4: Pool 模块开发

> **阶段目标**: 完整实现流动性池功能，包括池子列表、添加流动性、移除流动性
> **预计耗时**: 4-5 天

### 6.1 步骤 18: 创建 Pool 数据类型和查询

#### 作用说明
定义流动性池相关的数据类型和查询功能，这是 Pool 模块的数据基础。

#### 源码映射
- **参考目录**: `apps/web/src/lib/pool/`
- **参考文件**: 类型定义和数据获取

#### 依赖关系
- **依赖**: Phase 2 所有步骤
- **被依赖**: Pool UI 组件、流动性操作

#### 👉 自然语言代码提示词

```
请创建 Pool 数据类型和查询功能：

1. 创建 apps/web/src/types/pool.ts：
   - 定义 Pool 接口：
     * address: Address (池子合约地址)
     * token0: Token
     * token1: Token
     * reserve0: string (token0 储备量)
     * reserve1: string (token1 储备量)
     * totalSupply: string (LP 代币总量)
     * fee: number (手续费率，如 0.003 表示 0.3%)
     * version: 'v2' | 'v3' (池子版本)
     * chainId: number

   - 定义 Position 接口（用户持仓）：
     * pool: Pool
     * lpBalance: string (用户 LP 代币余额)
     * token0Share: string (用户占有的 token0 数量)
     * token1Share: string (用户占有的 token1 数量)
     * usdValue: number (持仓美元价值)
     * earnedFees: { token0: string, token1: string }

   - 定义 PoolStats 接口：
     * tvl: number (总锁仓价值)
     * volume24h: number (24小时交易量)
     * fees24h: number (24小时手续费)
     * apr: number (年化收益率)

2. 创建 apps/web/src/lib/pool/pool-fetcher.ts：
   - 获取池子列表：
     * fetchPools(chainId: number): Promise<Pool[]>

   - 获取单个池子：
     * fetchPool(address: Address, chainId: number): Promise<Pool | null>

   - 获取用户持仓：
     * fetchUserPositions(user: Address, chainId: number): Promise<Position[]>

   - 获取池子统计：
     * fetchPoolStats(pool: Pool): Promise<PoolStats>

3. 创建 apps/web/src/lib/hooks/usePools.ts：
   - 使用 useQuery 获取池子列表
   - 支持按代币筛选
   - 支持排序（TVL、APR、交易量）
   - 返回：
     * pools: Pool[]
     * isLoading: boolean
     * error: Error | null

4. 创建 apps/web/src/lib/hooks/usePool.ts：
   - 获取单个池子详情
   - 实时更新储备量
   - 返回 Pool 对象

5. 创建 apps/web/src/lib/hooks/useUserPositions.ts：
   - 获取用户所有持仓
   - 计算持仓价值
   - 计算累计收益
   - 返回 Position[]

6. 创建 apps/web/src/lib/pool/apr-calculator.ts：
   - 计算池子 APR：
     * calculateAPR(pool: Pool, stats: PoolStats): number

   - 考虑手续费收入和激励奖励
   - 返回年化百分比
```

---

### 6.2 步骤 19: 实现添加流动性逻辑

#### 作用说明
实现添加流动性的核心逻辑，计算需要存入的代币数量。

#### 源码映射
- **参考目录**: `apps/web/src/lib/pool/`
- **参考文件**: 流动性相关逻辑

#### 依赖关系
- **依赖**: 步骤 6.1
- **被依赖**: 添加流动性 UI

#### 👉 自然语言代码提示词

```
请实现添加流动性逻辑：

1. 创建 apps/web/src/lib/pool/liquidity/add-liquidity.ts：
   - 定义 AddLiquidityParams 接口：
     * token0: Token
     * token1: Token
     * amount0: string
     * amount1: string
     * slippage: number
     * deadline: number

   - 计算最优存入比例：
     * calculateOptimalAmounts(
         pool: Pool | null,
         token0: Token,
         token1: Token,
         amount0: string
       ): { amount0: string, amount1: string }

   - 如果是新池子（无流动性）：
     * 用户自由决定比例

   - 如果池子已有流动性：
     * 根据现有比例计算
     * amount1 = amount0 * reserve1 / reserve0

2. 创建 apps/web/src/lib/pool/liquidity/lp-tokens.ts：
   - 计算预计获得 LP 代币数量：
     * calculateLPTokens(
         pool: Pool,
         amount0: string,
         amount1: string
       ): string

   - 计算用户持仓占比：
     * calculatePoolShare(lpBalance: string, totalSupply: string): number

3. 创建 apps/web/src/lib/pool/transaction/add-liquidity-tx.ts：
   - 构建添加流动性交易：
     * buildAddLiquidityTx(params: AddLiquidityParams): TransactionRequest

   - 处理授权检查：
     * 检查两个代币是否都已授权
     * 返回需要授权的代币列表

   - 设置最小获得量（考虑滑点）

4. 创建 apps/web/src/lib/hooks/useAddLiquidity.ts：
   - 封装添加流动性流程：
     * calculateAmounts: (amount0: string) => { amount0, amount1 }
     * approveToken0: () => Promise<void>
     * approveToken1: () => Promise<void>
     * addLiquidity: () => Promise<Hash>
     * isCalculating: boolean
     * isApproving: boolean
     * isAdding: boolean
     * needsApproval0: boolean
     * needsApproval1: boolean
     * estimatedLPTokens: string

5. 创建 apps/web/src/lib/pool/price-range.ts（V3 用）：
   - 定义价格范围：
     * minPrice: number
     * maxPrice: number

   - 计算 tick 范围：
     * priceToTick(price: number): number
     * tickToPrice(tick: number): number

   - 验证价格范围有效性
```

---

### 6.3 步骤 20: 实现移除流动性逻辑

#### 作用说明
实现移除流动性的核心逻辑，计算能取回的代币数量。

#### 源码映射
- **参考目录**: `apps/web/src/lib/pool/`
- **参考文件**: 移除流动性相关逻辑

#### 依赖关系
- **依赖**: 步骤 6.1, 6.2
- **被依赖**: 移除流动性 UI

#### 👉 自然语言代码提示词

```
请实现移除流动性逻辑：

1. 创建 apps/web/src/lib/pool/liquidity/remove-liquidity.ts：
   - 定义 RemoveLiquidityParams 接口：
     * pool: Pool
     * lpAmount: string (要移除的 LP 代币数量)
     * slippage: number
     * deadline: number

   - 计算能取回的代币数量：
     * calculateRemoveLiquidityAmounts(
         pool: Pool,
         lpAmount: string
       ): { token0: string, token1: string }

   - 公式：
     * token0 = lpAmount / totalSupply * reserve0
     * token1 = lpAmount / totalSupply * reserve1

2. 创建 apps/web/src/lib/pool/liquidity/percent-selector.ts：
   - 百分比选项：25%, 50%, 75%, 100%
   - 计算对应 LP 数量：
     * calculateLPByPercent(
         balance: string,
         percent: number
       ): string

3. 创建 apps/web/src/lib/pool/transaction/remove-liquidity-tx.ts：
   - 构建移除流动性交易：
     * buildRemoveLiquidityTx(params: RemoveLiquidityParams): TransactionRequest

   - 设置最小获得量（考虑滑点）

4. 创建 apps/web/src/lib/hooks/useRemoveLiquidity.ts：
   - 封装移除流动性流程：
     * setPercent: (percent: number) => void
     * setCustomAmount: (amount: string) => void
     * calculateReceived: () => { token0, token1 }
     * removeLiquidity: () => Promise<Hash>
     * isRemoving: boolean
     * receivedTokens: { token0: string, token1: string }

5. 创建 apps/web/src/lib/pool/fees/claim-fees.ts：
   - 计算可领取的手续费：
     * calculateClaimableFees(position: Position): { token0, token1 }

   - 构建领取手续费交易
```

---

### 6.4 步骤 21: 创建 Pool 列表组件

#### 作用说明
创建展示所有流动性池的列表组件，用户可以浏览和选择池子。

#### 源码映射
- **参考目录**: `apps/web/src/app/(networks)/(evm)/[chainId]/pool/`
- **参考文件**: Pool 列表相关组件

#### 依赖关系
- **依赖**: 步骤 6.1
- **被依赖**: Pool 主页面

#### 👉 自然语言代码提示词

```
请创建 Pool 列表组件：

1. 创建 apps/web/src/components/pool/PoolList.tsx：
   - 组件属性：
     * pools: Pool[]
     * isLoading: boolean
     * onPoolSelect: (pool: Pool) => void

   - 列表结构：
     * 表头：代币对、TVL、交易量、APR、操作
     * 表体：池子行列表

   - 功能：
     * 点击行展开详情
     * 排序（点击表头）
     * 搜索过滤
     * 分页或虚拟列表

2. 创建 apps/web/src/components/pool/PoolRow.tsx：
   - 单行池子信息：
     * 代币对图标（两个代币图标并排）
     * 代币对名称（USDC/ETH）
     * 手续费率标签（0.3%）
     * TVL（格式化显示）
     * 24h 交易量
     * APR（带颜色标识）
     * 操作按钮（添加流动性）

   - 展开后显示：
     * 储备量详情
     * 用户持仓（如果有）
     * 快速操作按钮

3. 创建 apps/web/src/components/pool/PoolFilters.tsx：
   - 筛选选项：
     * 搜索框（按代币搜索）
     * 版本筛选（V2/V3/全部）
     * 我的持仓开关
     * 排序下拉菜单

   - 实时过滤列表

4. 创建 apps/web/src/components/pool/PoolStatsCard.tsx：
   - 统计卡片：
     * 总 TVL
     * 24h 总交易量
     * 24h 总手续费
     * 池子数量

   - 大数字格式化显示
   - 变化趋势指示

5. 创建 apps/web/src/components/pool/EmptyPoolList.tsx：
   - 空列表状态
   - 提示信息
   - 创建池子引导（可选）

6. 创建 apps/web/src/components/pool/PoolSkeleton.tsx：
   - 加载骨架屏
   - 多行占位效果
```

---

### 6.5 步骤 22: 创建添加流动性组件

#### 作用说明
创建添加流动性的完整界面，包括代币选择、数量输入、预览等。

#### 源码映射
- **参考目录**: `apps/web/src/app/(networks)/(evm)/[chainId]/pool/`
- **参考文件**: 添加流动性相关组件

#### 依赖关系
- **依赖**: 步骤 6.2
- **被依赖**: Pool 页面

#### 👉 自然语言代码提示词

```
请创建添加流动性组件：

1. 创建 apps/web/src/components/pool/AddLiquidityWidget.tsx：
   - 主要组件，组合所有子组件

   - 状态：
     * token0, token1
     * amount0, amount1
     * pool: Pool | null (可能已存在或新建)
     * isPreviewOpen

   - 处理函数：
     * handleToken0Select
     * handleToken1Select
     * handleAmount0Change (自动计算 amount1)
     * handleAmount1Change (自动计算 amount0)
     * handleAddLiquidity

2. 创建 apps/web/src/components/pool/TokenPairSelector.tsx：
   - 代币对选择器
   - 两个代币选择按钮
   - 交换位置按钮
   - 热门代币对快捷选择

3. 创建 apps/web/src/components/pool/LiquidityAmountInput.tsx：
   - 流动性数量输入
   - 双输入框（两个代币）
   - 显示余额
   - 显示比例提示
   - 最大按钮

4. 创建 apps/web/src/components/pool/PoolRatioDisplay.tsx：
   - 显示当前池子比例
   - 价格比率（1 ETH = 2000 USDC）
   - 用户存入比例
   - 比例匹配状态（是否按池子比例存入）

5. 创建 apps/web/src/components/pool/AddLiquidityPreview.tsx：
   - 添加流动性预览：
     * 存入代币详情
     * 预计获得 LP 代币数量
     * 池子份额占比
     * 滑点设置
     * 确认按钮

6. 创建 apps/web/src/components/pool/LPPositionSummary.tsx：
   - 添加后的持仓摘要
   - 显示新持仓
   - 查看持仓链接
   - 继续添加按钮

7. 创建 apps/web/src/components/pool/PriceRangeSelector.tsx（V3）：
   - 价格范围选择
   - 最小/最大价格输入
   - 全范围选项
   - 价格范围图表（可选）
```

---

### 6.6 步骤 23: 创建移除流动性组件

#### 作用说明
创建移除流动性的界面，允许用户选择移除比例并预览结果。

#### 源码映射
- **参考目录**: `apps/web/src/app/(networks)/(evm)/[chainId]/pool/`
- **参考文件**: 移除流动性相关组件

#### 依赖关系
- **依赖**: 步骤 6.3
- **被依赖**: 持仓管理页面

#### 👉 自然语言代码提示词

```
请创建移除流动性组件：

1. 创建 apps/web/src/components/pool/RemoveLiquidityWidget.tsx：
   - 主要组件

   - 属性：
     * position: Position (用户持仓)

   - 状态：
     * percent: number (移除百分比)
     * customAmount: string (自定义数量)
     * isPreviewOpen

   - 处理函数：
     * handlePercentSelect
     * handleCustomAmountChange
     * handleRemoveLiquidity

2. 创建 apps/web/src/components/pool/RemovePercentButtons.tsx：
   - 百分比选择按钮组
   - 25%, 50%, 75%, Max
   - 选中状态样式
   - 自定义输入选项

3. 创建 apps/web/src/components/pool/RemoveLiquidityPreview.tsx：
   - 移除预览：
     * 移除 LP 数量
     * 预计获得 token0 数量
     * 预计获得 token1 数量
     * 当前价格
     * 滑点设置
     * 确认按钮

4. 创建 apps/web/src/components/pool/PositionCard.tsx：
   - 持仓卡片：
     * 代币对信息
     * LP 余额
     * 持仓价值
     * 占比
     * 累计收益
     * 操作按钮（添加/移除）

5. 创建 apps/web/src/components/pool/UserPositionsList.tsx：
   - 用户持仓列表
   - 多个 PositionCard
   - 总价值汇总
   - 空状态处理

6. 创建 apps/web/src/components/pool/ClaimFeesButton.tsx：
   - 领取手续费按钮
   - 显示可领取金额
   * 一键领取
```

---

### 6.7 步骤 24: 组装 Pool 主页面

#### 作用说明
将所有 Pool 组件组装成完整的页面，实现池子浏览和流动性管理。

#### 源码映射
- **参考目录**: `apps/web/src/app/(networks)/(evm)/[chainId]/pool/`
- **参考文件**: `page.tsx`, `layout.tsx`

#### 依赖关系
- **依赖**: 步骤 6.4, 6.5, 6.6
- **被依赖**: 无（这是最终页面）

#### 👉 自然语言代码提示词

```
请组装 Pool 主页面：

1. 创建 apps/web/src/app/[chainId]/pool/page.tsx：
   - Pool 主页面组件

   - 页面结构：
     * 页面标题和描述
     * 统计卡片（PoolStatsCard）
     * 筛选器（PoolFilters）
     * 池子列表（PoolList）
     * 添加流动性按钮（跳转到添加页面）

   - 数据获取：
     * 使用 usePools 获取池子列表
     * 使用 useUserPositions 获取用户持仓

   - 状态管理：
     * 筛选条件
     * 排序方式
     * 选中的池子

2. 创建 apps/web/src/app/[chainId]/pool/layout.tsx：
   - 页面布局
   - 子导航（池子列表/我的持仓）
   - 页面标题

3. 创建 apps/web/src/app/[chainId]/pool/add/page.tsx：
   - 添加流动性页面
   - 渲染 AddLiquidityWidget
   - 从 URL 参数预填代币

4. 创建 apps/web/src/app/[chainId]/pool/remove/page.tsx：
   - 移除流动性页面
   - 渲染 RemoveLiquidityWidget
   - 从 URL 参数获取池子地址

5. 创建 apps/web/src/app/[chainId]/pool/positions/page.tsx：
   - 我的持仓页面
   - 渲染 UserPositionsList
   - 显示总价值汇总
   - 快速操作入口

6. 创建 apps/web/src/components/pool/PoolNavigation.tsx：
   - 子导航组件
   - 链接：池子列表 / 我的持仓
   - 当前页面高亮
   - 添加流动性按钮

7. 创建 apps/web/src/components/pool/PoolHeader.tsx：
   - 页面头部
   - 标题："流动性池"
   - 说明文字
   - 帮助链接
```

---

## 七、Phase 5: 集成与测试

> **阶段目标**: 完成模块集成、端到端测试、性能优化
> **预计耗时**: 2-3 天

### 7.1 步骤 25: 创建共享 Hooks

#### 作用说明
创建可复用的 React Hooks，简化组件逻辑，提高代码复用性。

#### 源码映射
- **参考目录**: `apps/web/src/lib/hooks/`, `packages/hooks/`
- **参考文件**: 各种自定义 hooks

#### 依赖关系
- **依赖**: 所有前置步骤
- **被依赖**: 所有组件

#### 👉 自然语言代码提示词

```
请创建共享 Hooks：

1. 创建 apps/web/src/lib/hooks/useDebounce.ts：
   - 防抖 hook
   - 参数：value, delay
   - 用于输入框延迟处理

2. 创建 apps/web/src/lib/hooks/useLocalStorage.ts：
   - localStorage 状态同步
   - 参数：key, initialValue
   - 返回：[value, setValue]

3. 创建 apps/web/src/lib/hooks/useMediaQuery.ts：
   - 响应式断点检测
   - 参数：query
   - 返回：boolean

4. 创建 apps/web/src/lib/hooks/useCopyToClipboard.ts：
   - 复制到剪贴板
   - 返回：copy 函数和 copied 状态

5. 创建 apps/web/src/lib/hooks/useBlockNumber.ts：
   - 获取当前区块号
   - 实时更新
   - 用于刷新数据

6. 创建 apps/web/src/lib/hooks/useTransaction.ts：
   - 交易状态管理
   - 参数：txHash
   - 返回：状态、确认数、收据

7. 创建 apps/web/src/lib/hooks/useTokenAllowance.ts：
   - 查询代币授权额度
   - 参数：token, owner, spender
   - 返回：授权额度

8. 创建 apps/web/src/lib/hooks/useApprove.ts：
   - 封装授权流程
   - 返回：approve 函数、状态

9. 创建 apps/web/src/lib/hooks/useChainId.ts：
   - 获取当前链 ID
   - 处理链切换

10. 创建 apps/web/src/lib/hooks/useAccount.ts：
    - 获取当前账户信息
    - 返回地址、连接状态等
```

---

### 7.2 步骤 26: 实现错误处理和通知系统

#### 作用说明
建立统一的错误处理和用户通知系统，提升用户体验。

#### 源码映射
- **参考目录**: `apps/web/src/lib/notifications/`, `packages/notifications/`
- **参考文件**: 通知相关组件

#### 依赖关系
- **依赖**: 所有前置步骤
- **被依赖**: 所有需要通知的功能

#### 👉 自然语言代码提示词

```
请实现错误处理和通知系统：

1. 创建 apps/web/src/lib/notifications/toast-store.ts：
   - 使用 Zustand 创建 toast store
   - 状态：toasts 数组
   - 操作：
     * addToast(toast)
     * removeToast(id)
     * clearAll()

   - Toast 类型：
     * id: string
     * type: 'success' | 'error' | 'warning' | 'info'
     * title: string
     * description?: string
     * duration?: number
     * action?: { label, onClick }

2. 创建 apps/web/src/components/ui/Toast.tsx：
   - 单个 toast 组件
   - 根据类型显示不同图标和颜色
   - 进度条（自动关闭）
   - 关闭按钮
   - 操作按钮（如果有）

3. 创建 apps/web/src/components/ui/ToastContainer.tsx：
   - Toast 容器
   - 管理多个 toast 的位置
   - 堆叠效果
   - 动画（进入/退出）

4. 创建 apps/web/src/lib/notifications/useToast.ts：
   - 封装 toast 操作
   - 提供便捷方法：
     * toast.success(title, description)
     * toast.error(title, description)
     * toast.warning(title, description)
     * toast.info(title, description)

5. 创建 apps/web/src/lib/error/error-handler.ts：
   - 统一错误处理
   - 解析区块链错误
   * 用户拒绝签名
   * gas 不足
   * 交易失败
   * 网络错误

   - 转换为友好错误信息

6. 创建 apps/web/src/components/error/ErrorBoundary.tsx：
   - React 错误边界
   - 捕获渲染错误
   - 显示错误页面
   - 重试按钮

7. 创建 apps/web/src/components/error/ErrorPage.tsx：
   - 错误页面组件
   - 错误图标
   - 错误信息
   - 返回首页按钮
   * 联系支持链接

8. 创建 apps/web/src/app/error.tsx：
   - Next.js 错误页面
   - 使用 ErrorPage 组件
```

---

### 7.3 步骤 27: 添加加载状态和骨架屏

#### 作用说明
优化加载体验，使用骨架屏和加载状态减少用户等待焦虑。

#### 源码映射
- **参考目录**: `apps/web/src/components/ui/`
- **参考文件**: 骨架屏组件

#### 依赖关系
- **依赖**: 所有前置步骤
- **被依赖**: 所有需要加载状态的组件

#### 👉 自然语言代码提示词

```
请添加加载状态和骨架屏：

1. 创建 apps/web/src/components/ui/Skeleton.tsx：
   - 基础骨架屏组件
   - 支持不同形状：
     * 矩形（默认）
     * 圆形
     * 文本行

   - 动画效果（脉冲或波浪）
   - 可配置宽高

2. 创建 apps/web/src/components/ui/Spinner.tsx：
   - 加载旋转图标
   - 可配置大小和颜色
   - 支持在按钮内使用

3. 创建 apps/web/src/components/swap/SwapWidgetSkeleton.tsx：
   - Swap 组件骨架屏
   - 模拟输入框、按钮布局
   - 整体占位效果

4. 创建 apps/web/src/components/pool/PoolListSkeleton.tsx：
   - 池子列表骨架屏
   - 多行占位
   - 表头占位

5. 创建 apps/web/src/components/ui/LoadingOverlay.tsx：
   - 加载遮罩层
   - 半透明背景
   - 居中加载图标
   - 可选加载文字

6. 创建 apps/web/src/components/ui/AsyncButton.tsx：
   - 异步操作按钮
   - 自动处理 loading 状态
   - 显示加载图标
   - 禁用点击

7. 更新现有组件添加骨架屏：
   - TokenSelector 骨架屏
   - PoolRow 骨架屏
   - PositionCard 骨架屏

8. 创建 apps/web/src/app/loading.tsx：
   - Next.js 全局加载页面
   - 显示应用加载状态
```

---

### 7.4 步骤 28: 实现响应式设计

#### 作用说明
确保应用在各种设备上都能良好显示，特别是移动端适配。

#### 源码映射
- **参考文件**: Tailwind 配置，组件样式

#### 依赖关系
- **依赖**: 所有前置步骤
- **被依赖**: 无

#### 👉 自然语言代码提示词

```
请实现响应式设计：

1. 更新 apps/web/tailwind.config.js：
   - 确认断点配置：
     * sm: 640px
     * md: 768px
     * lg: 1024px
     * xl: 1280px
     * 2xl: 1536px

2. 创建响应式布局组件：
   - apps/web/src/components/layout/Container.tsx
     * 响应式最大宽度
     * 响应式内边距

   - apps/web/src/components/layout/ResponsiveGrid.tsx
     * 移动端单列
     * 平板双列
     * 桌面三列+

3. 更新 Swap 组件响应式：
   - SwapWidget 宽度适配
   - TokenSelector 全屏弹窗（移动端）
   - 按钮大小调整
   - 字体大小调整

4. 更新 Pool 组件响应式：
   - PoolList 横向滚动（小屏幕）
   - PoolRow 卡片化（移动端）
   - 筛选器折叠（移动端）

5. 创建移动端导航：
   - apps/web/src/components/layout/MobileNav.tsx
     * 底部导航栏
     * 图标 + 文字
     * 当前页面高亮

6. 创建移动端菜单：
   - apps/web/src/components/layout/MobileMenu.tsx
     * 汉堡菜单
     * 侧边抽屉
     * 导航链接

7. 测试响应式断点：
   - 320px (小手机)
   - 375px (iPhone SE)
   - 414px (iPhone Max)
   - 768px (iPad)
   - 1024px (iPad Pro)
   - 1440px (桌面)

8. 添加触摸优化：
   - 按钮最小点击区域 44px
   - 滑动手势支持
   - 下拉刷新（可选）
```

---

### 7.5 步骤 29: 编写单元测试

#### 作用说明
为核心逻辑编写单元测试，确保代码质量和可维护性。

#### 源码映射
- **参考目录**: `apps/web/test/`, `packages/ui/test/`
- **参考文件**: 测试文件

#### 依赖关系
- **依赖**: 所有前置步骤
- **被依赖**: 无

#### 👉 自然语言代码提示词

```
请编写单元测试：

1. 配置测试环境：
   - 安装依赖：
     * vitest
     * @testing-library/react
     * @testing-library/jest-dom
     * jsdom
     * @vitejs/plugin-react

   - 创建 vitest.config.ts
   - 配置测试脚本

2. 测试工具函数：
   - apps/web/src/lib/__tests__/functions.test.ts
     * 测试 formatBalance
     * 测试 parseBalance
     * 测试 calculatePriceImpact
     * 测试 formatPrice

3. 测试 Hooks：
   - apps/web/src/lib/hooks/__tests__/useDebounce.test.ts
   - apps/web/src/lib/hooks/__tests__/useLocalStorage.test.ts
   - 使用 renderHook 测试

4. 测试组件：
   - apps/web/src/components/ui/__tests__/Button.test.tsx
   - apps/web/src/components/swap/__tests__/TokenInput.test.tsx
   - 使用 render + screen 测试
   - 测试交互行为

5. 测试 Store：
   - apps/web/src/lib/store/__tests__/settings-store.test.ts
   - 测试状态更新
   - 测试持久化

6. 测试 Swap 逻辑：
   - apps/web/src/lib/swap/__tests__/routing.test.ts
     * 测试路径计算
     * 测试价格影响计算

   - apps/web/src/lib/swap/__tests__/transaction.test.ts
     * 测试交易构建
     * 测试授权检查

7. 测试 Pool 逻辑：
   - apps/web/src/lib/pool/__tests__/liquidity.test.ts
     * 测试 LP 代币计算
     * 测试移除流动性计算

8. 创建测试工具：
   - apps/web/test/utils.tsx
     * 创建测试渲染函数
     * 包装所有 Provider

   - apps/web/test/mocks/
     * 模拟数据
     * 模拟 API 响应

9. 配置 CI 测试：
   - 更新 .github/workflows/test.yml
   - 运行测试命令
   - 生成覆盖率报告
```

---

### 7.6 步骤 30: 编写集成测试和 E2E 测试

#### 作用说明
编写端到端测试，验证完整用户流程是否正常工作。

#### 源码映射
- **参考目录**: `apps/web/test/e2e/`
- **参考文件**: E2E 测试文件

#### 依赖关系
- **依赖**: 步骤 7.5
- **被依赖**: 无

#### 👉 自然语言代码提示词

```
请编写集成测试和 E2E 测试：

1. 配置 Playwright：
   - 安装 @playwright/test
   - 创建 playwright.config.ts
   - 配置测试浏览器（Chromium, Firefox, WebKit）
   - 配置测试环境变量

2. 创建测试工具：
   - apps/web/test/e2e/utils/wallet-mock.ts
     * 模拟钱包连接
     * 模拟交易签名

   - apps/web/test/e2e/utils/test-setup.ts
     * 测试前置条件
     * 页面初始化

3. 编写 Swap E2E 测试：
   - apps/web/test/e2e/swap/swap-flow.spec.ts
     * 测试页面加载
     * 测试代币选择
     * 测试金额输入
     * 测试报价计算
     * 测试交易执行（模拟）
     * 测试交易成功提示

4. 编写 Pool E2E 测试：
   - apps/web/test/e2e/pool/pool-list.spec.ts
     * 测试池子列表加载
     * 测试筛选功能
     * 测试排序功能

   - apps/web/test/e2e/pool/add-liquidity.spec.ts
     * 测试添加流动性流程
     * 测试代币对选择
     * 测试数量计算

   - apps/web/test/e2e/pool/remove-liquidity.spec.ts
     * 测试移除流动性流程
     * 测试百分比选择

5. 编写导航测试：
   - apps/web/test/e2e/navigation.spec.ts
     * 测试页面跳转
     * 测试导航菜单
     * 测试移动端导航

6. 编写钱包连接测试：
   - apps/web/test/e2e/wallet/connect.spec.ts
     * 测试连接按钮
     * 测试钱包选择
     * 测试连接状态显示

7. 配置测试数据：
   - apps/web/test/e2e/fixtures/tokens.json
     * 测试用代币数据

   - apps/web/test/e2e/fixtures/pools.json
     * 测试用池子数据

8. 配置 CI/CD：
   - 添加 E2E 测试工作流
   - 配置测试报告上传
   - 配置失败截图上传
```

---

### 7.7 步骤 31: 性能优化

#### 作用说明
优化应用性能，提升加载速度和运行效率。

#### 源码映射
- **参考文件**: Next.js 配置，组件代码

#### 依赖关系
- **依赖**: 所有前置步骤
- **被依赖**: 无

#### 👉 自然语言代码提示词

```
请进行性能优化：

1. 代码分割优化：
   - 使用 dynamic import 懒加载：
     * TokenSelector 弹窗
     * 图表组件
     * 大型依赖库

   - 配置 Next.js 代码分割策略

2. 图片优化：
   - 使用 Next.js Image 组件
   - 配置代币图标 CDN
   - 使用 WebP 格式
   - 配置占位符

3. 数据缓存优化：
   - 配置 TanStack Query 缓存策略
   - 设置合理的 staleTime
   - 使用 prefetch
   - 实现乐观更新

4. 渲染优化：
   - 使用 React.memo 避免不必要重渲染
   - 使用 useMemo 缓存计算结果
   - 使用 useCallback 缓存回调函数
   - 虚拟列表（大量数据时）

5. 网络优化：
   - 启用 HTTP/2
   - 配置 CDN
   - 压缩响应
   - 使用 Service Worker 缓存（可选）

6. 构建优化：
   - 分析打包体积
   - 移除未使用的依赖
   - 配置 tree shaking
   - 启用 gzip/brotli 压缩

7. 运行时优化：
   - 减少重渲染
   - 防抖/节流高频事件
   - 使用 Web Workers（复杂计算）

8. 监控和测量：
   - 配置 Vercel Analytics
   - 添加性能指标收集
   - 监控 Core Web Vitals
```

---

## 八、开发顺序总结

### 推荐执行顺序

```
Phase 1: 项目初始化（步骤 1-5）
    │
    ▼
Phase 2: 核心数据层（步骤 6-10）
    │
    ├──▶ 可以并行开发 Swap 和 Pool
    │
    ▼
Phase 3: Swap 模块（步骤 11-17）
    │
    ├──▶ 步骤 11-12: 核心逻辑
    ├──▶ 步骤 13-16: UI 组件（可并行）
    └──▶ 步骤 17: 页面组装
    │
    ▼
Phase 4: Pool 模块（步骤 18-24）
    │
    ├──▶ 步骤 18-20: 核心逻辑
    ├──▶ 步骤 21-23: UI 组件（可并行）
    └──▶ 步骤 24: 页面组装
    │
    ▼
Phase 5: 集成与测试（步骤 25-31）
    │
    ├──▶ 步骤 25-28: 通用功能
    └──▶ 步骤 29-31: 测试优化
```

### 依赖关系图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            开发依赖关系                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  项目初始化 ─────────────────────────────────────────────────────────────▶ │
│       │                                                                     │
│       ▼                                                                     │
│  核心数据层 ─────────────────────────────────────────────────────────────▶ │
│       │                                                                     │
│       ├──▶ Swap 路由计算 ──▶ Swap 交易构建 ──▶ Swap UI ──▶ Swap 页面     │
│       │                                                                     │
│       └──▶ Pool 数据查询 ──▶ 流动性逻辑 ──▶ Pool UI ──▶ Pool 页面        │
│                                                                             │
│  通用功能（Hooks、通知、错误处理）◀──────────────────────────────────────────│
│       │                                                                     │
│       ▼                                                                     │
│  测试与优化 ◀──────────────────────────────────────────────────────────────│
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 九、关键技术要点

### 9.1 Swap 核心算法

```
路由计算（简化版）：

1. 获取所有可能的池子
2. 构建图：代币为节点，池子为边
3. 使用 Dijkstra 或类似算法找最优路径
4. 考虑：
   - 输出金额最大化
   - gas 成本
   - 价格影响
```

### 9.2 流动性计算

```
添加流动性：
- 新池子：自由比例
- 已有池子：amount1 = amount0 * reserve1 / reserve0

LP 代币计算：
- 新池子：sqrt(amount0 * amount1)
- 已有池子：amount0 * totalSupply / reserve0

移除流动性：
- token0 = lpAmount * reserve0 / totalSupply
- token1 = lpAmount * reserve1 / totalSupply
```

### 9.3 滑点保护

```
最小获得量 = 预期输出 * (1 - 滑点容忍度)

例如：
- 预期获得 100 USDC
- 滑点容忍度 0.5%
- 最小获得量 = 100 * 0.995 = 99.5 USDC
```

---

## 十、常见问题与解决方案

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 报价计算慢 | 需要查询多个池子 | 使用缓存、批量请求、乐观更新 |
| 交易失败 | gas 不足或滑点过高 | 增加 gas limit、提高滑点容忍度 |
| 授权频繁 | 每次都需要授权 | 使用无限授权选项 |
| 价格延迟 | 链上数据变化快 | 使用 WebSocket 实时更新 |
| 移动端体验差 | 布局不适配 | 响应式设计、触摸优化 |

---

## 十一、资源与参考

### 官方文档
- [Wagmi Documentation](https://wagmi.sh/)
- [Viem Documentation](https://viem.sh/)
- [Next.js Documentation](https://nextjs.org/docs)
- [TanStack Query](https://tanstack.com/query/latest)

### SushiSwap 参考
- [GitHub Repository](https://github.com/sushi-labs/sushiswap)
- [Smart Contracts](https://github.com/sushi-labs/sushiswap/tree/master/protocols)
- [SDK Documentation](https://docs.sushi.com/)

### 学习资源
- [Uniswap V2 Whitepaper](https://uniswap.org/whitepaper.pdf)
- [Uniswap V3 Whitepaper](https://uniswap.org/whitepaper-v3.pdf)
- [Ethereum Development Guide](https://ethereum.org/en/developers/)

---

## 十二、附录：完整文件清单

### 配置文件
```
├── package.json
├── pnpm-workspace.yaml
├── turbo.json
├── biome.json
├── tsconfig.json
└── .gitignore
```

### 源代码文件（核心）
```
apps/web/src/
├── app/
│   ├── layout.tsx
│   ├── page.tsx
│   ├── globals.css
│   ├── [chainId]/
│   │   ├── swap/
│   │   │   ├── page.tsx
│   │   │   └── layout.tsx
│   │   └── pool/
│   │       ├── page.tsx
│   │       ├── layout.tsx
│   │       ├── add/
│   │       │   └── page.tsx
│   │       └── remove/
│   │           └── page.tsx
│   ├── error.tsx
│   └── loading.tsx
│
├── components/
│   ├── ui/                    # 基础 UI 组件
│   ├── swap/                  # Swap 组件
│   └── pool/                  # Pool 组件
│
├── lib/
│   ├── constants.ts
│   ├── network.ts
│   ├── functions.ts
│   │
│   ├── wagmi/
│   │   └── config.ts
│   │
│   ├── hooks/
│   │   ├── useTokens.ts
│   │   ├── useTokenPrice.ts
│   │   ├── useBalance.ts
│   │   ├── useSwapQuote.ts
│   │   ├── useSwap.ts
│   │   ├── usePools.ts
│   │   ├── useAddLiquidity.ts
│   │   └── useRemoveLiquidity.ts
│   │
│   ├── store/
│   │   ├── settings-store.ts
│   │   ├── swap-store.ts
│   │   └── pool-store.ts
│   │
│   ├── swap/
│   │   ├── types.ts
│   │   ├── routing/
│   │   │   ├── router.ts
│   │   │   ├── price-impact.ts
│   │   │   └── gas-estimator.ts
│   │   ├── transaction/
│   │   │   ├── builder.ts
│   │   │   ├── approver.ts
│   │   │   └── sender.ts
│   │   └── api-base-url.ts
│   │
│   ├── pool/
│   │   ├── pool-fetcher.ts
│   │   ├── apr-calculator.ts
│   │   └── liquidity/
│   │       ├── add-liquidity.ts
│   │       ├── remove-liquidity.ts
│   │       └── lp-tokens.ts
│   │
│   ├── tokens/
│   │   ├── token-list.ts
│   │   └── token-cache.ts
│   │
│   ├── prices/
│   │   ├── price-api.ts
│   │   └── price-utils.ts
│   │
│   └── notifications/
│       ├── toast-store.ts
│       └── useToast.ts
│
├── providers/
│   ├── providers.tsx
│   ├── wagmi-provider.tsx
│   └── query-provider.tsx
│
└── types/
    ├── token.ts
    └── pool.ts
```

---

**文档结束**

> 本计划书提供了从零开始复刻 SushiSwap Swap 和 Pool 模块的完整指南。按照步骤逐步执行，即可完成一个功能完整的 DEX 前端应用。
> 
> 建议在实际开发中：
> 1. 先完成 Phase 1 和 Phase 2 的基础建设
> 2. 然后可以选择先实现 Swap 或 Pool（两者相对独立）
> 3. 最后进行集成测试和优化
> 4. 每个步骤完成后进行测试验证
