apps/app/
│
├── 📄 配置文件
│   ├── package.json                    # 依賴管理
│   ├── next.config.js                  # Next.js 配置
│   ├── tailwind.config.js              # Tailwind CSS 配置
│   ├── tsconfig.json                   # TypeScript 配置
│   ├── postcss.config.js               # PostCSS 配置
│   ├── vercel.json                     # Vercel 部署配置
│   ├── .env.example                    # 環境變量示例
│   ├── .env.local                      # 本地環境變量
│   ├── global.d.ts                     # 全局類型定義
│   └── README.md                       # 項目說明文檔
│
├── 📁 messages/                        # 多語言翻譯文件 (i18n)
│   ├── en.json                         # 英語
│   ├── zh.json                         # 中文
│   ├── es.json                         # 西班牙語
│   ├── fr.json                         # 法語
│   ├── de.json                         # 德語
│   ├── it.json                         # 意大利語
│   ├── pt.json                         # 葡萄牙語
│   ├── ru.json                         # 俄語
│   ├── uk.json                         # 烏克蘭語
│   ├── tr.json                         # 土耳其語
│   ├── nl.json                         # 荷蘭語
│   ├── ko.json                         # 韓語
│   ├── hi.json                         # 印地語
│   └── fil.json                        # 菲律賓語
│
├── 📁 public/                          # 靜態資源
│   ├── favicon.ico                     # 網站圖標
│   ├── favicon.png
│   ├── manifest.json                   # PWA 配置
│   ├── robots.txt                      # 搜索引擎爬蟲規則
│   ├── sitemap.xml                     # 網站地圖
│   ├── sw.js                           # Service Worker
│   │
│   ├── 📁 icons/                       # 各種代幣和網絡圖標
│   │   ├── przUSDC.svg                 # 各種 Prize 代幣圖標
│   │   ├── przWETH.svg
│   │   ├── przDAI.svg
│   │   ├── mainnet.svg                 # 主網圖標
│   │   ├── optimism.svg                # Optimism 圖標
│   │   ├── arbitrum.svg                # Arbitrum 圖標
│   │   └── ... (50+ 代幣和網絡圖標)
│   │
│   ├── 📁 fonts/                       # 字體文件
│   │   ├── inter/                      # Inter 字體系列
│   │   │   ├── inter-regular.woff
│   │   │   ├── inter-bold.woff
│   │   │   └── ... (7個字重)
│   │   └── grotesk/                    # Founders Grotesk 字體
│   │       ├── founders-grotesk-regular.woff2
│   │       ├── founders-grotesk-bold.woff2
│   │       └── founders-grotesk-medium.woff2
│   │
│   ├── 📁 .well-known/                 # 協議配置
│   │   └── farcaster.json              # Farcaster 社交協議
│   │
│   └── 其他圖片資源
│       ├── pooltogether-logo.svg       # Logo
│       ├── checkingPrizesSpinner.gif   # 加載動畫
│       └── ... (社交分享圖片等)
│
└── 📁 src/                             # 源代碼目錄
    │
    ├── 📁 app/                         # Next.js App Router (API 路由)
    │   └── api/                        # API 端點
    │       │
    │       ├── 📁 frame/               # Farcaster Frame API
    │       │   ├── utils.ts
    │       │   ├── components.tsx
    │       │   ├── images.tsx
    │       │   └── account/
    │       │       ├── route.ts
    │       │       ├── image/route.tsx
    │       │       └── [user]/
    │       │           ├── route.ts
    │       │           └── image/route.tsx
    │       │
    │       ├── 📁 prizes/              # 獎品相關 API
    │       │   ├── route.ts
    │       │   └── [chainId]/
    │       │       ├── route.ts
    │       │       └── utils.ts
    │       │
    │       ├── 📁 vault/               # 金庫相關 API
    │       │   ├── route.ts
    │       │   └── [chainId]/
    │       │       ├── route.ts
    │       │       └── [vaultAddress]/
    │       │           ├── route.ts
    │       │           └── utils.ts
    │       │
    │       └── 📁 vaultList/           # 金庫列表 API
    │           ├── route.ts
    │           ├── default/route.ts
    │           └── meme/route.ts
    │
    ├── 📁 pages/                       # Next.js 頁面 (Pages Router)
    │   ├── _app.tsx                    # App 根組件 (WAGMI/Provider 配置)
    │   ├── _document.tsx               # HTML Document 配置
    │   ├── 404.tsx                     # 404 錯誤頁面
    │   ├── index.tsx                   # 首頁
    │   ├── vaults.tsx                  # 金庫列表頁
    │   ├── prizes.tsx                  # 獎品頁面
    │   │
    │   ├── 📁 account/                 # 帳戶相關頁面
    │   │   ├── index.tsx               # 當前用戶帳戶頁
    │   │   └── [user].tsx              # 其他用戶帳戶頁（動態路由）
    │   │
    │   └── 📁 vault/                   # 金庫詳情頁面
    │       └── [chainId]/
    │           └── [vaultAddress].tsx  # 特定金庫頁面
    │
    ├── 📁 components/                  # React 組件
    │   ├── Layout.tsx                  # 全局佈局
    │   ├── Navbar.tsx                  # 導航欄
    │   ├── Footer.tsx                  # 頁腳
    │   ├── HomeHeader.tsx              # 首頁頭部
    │   ├── AppContainer.tsx            # App 容器
    │   ├── CopyButton.tsx              # 複製按鈕
    │   ├── PageNotFound.tsx            # 404 組件
    │   ├── VaultListHandler.tsx        # 金庫列表處理器
    │   ├── TemporaryMoonwellWarning.tsx # 臨時警告
    │   │
    │   ├── 📁 Frames/                  # Farcaster Frames
    │   │   ├── DefaultFrame.tsx
    │   │   ├── AccountFrame.tsx
    │   │   └── VaultFrame.tsx
    │   │
    │   ├── 📁 Modals/                  # 彈窗組件 ⭐ 核心交互
    │   │   ├── TxFormInput.tsx         # 交易表單輸入
    │   │   ├── Odds.tsx                # 中獎概率顯示
    │   │   ├── NetworkFees.tsx         # 網絡費用顯示
    │   │   │
    │   │   ├── 📁 DepositModal/        # 存款彈窗 💰
    │   │   │   ├── index.tsx
    │   │   │   ├── DepositForm.tsx
    │   │   │   ├── DepositTxButton.tsx
    │   │   │   ├── DepositZapTxButton.tsx
    │   │   │   └── Views/
    │   │   │       ├── MainView.tsx
    │   │   │       ├── ReviewView.tsx
    │   │   │       ├── WaitingView.tsx
    │   │   │       ├── ConfirmingView.tsx
    │   │   │       ├── SuccessView.tsx
    │   │   │       └── ErrorView.tsx
    │   │   │
    │   │   ├── 📁 WithdrawModal/       # 取款彈窗 💸
    │   │   │   ├── index.tsx
    │   │   │   ├── WithdrawForm.tsx
    │   │   │   ├── WithdrawTxButton.tsx
    │   │   │   ├── WithdrawZapTxButton.tsx
    │   │   │   └── Views/ (6個視圖)
    │   │   │
    │   │   ├── 📁 DelegateModal/       # 委託彈窗 🎯
    │   │   │   ├── index.tsx
    │   │   │   ├── DelegateForm.tsx
    │   │   │   ├── DelegateTxButton.tsx
    │   │   │   └── DelegateModalBody.tsx
    │   │   │
    │   │   ├── 📁 CheckPrizesModal/    # 檢查獎品彈窗 🎁
    │   │   │   ├── index.tsx
    │   │   │   ├── animations.ts
    │   │   │   └── Views/
    │   │   │       ├── CheckingView.tsx
    │   │   │       ├── WinView.tsx
    │   │   │       └── NoWinView.tsx
    │   │   │
    │   │   ├── 📁 DrawModal/           # 開獎彈窗
    │   │   │   ├── index.tsx
    │   │   │   └── Views/
    │   │   │       └── MainView.tsx
    │   │   │
    │   │   └── 📁 SettingsModal/       # 設置彈窗 ⚙️
    │   │       ├── index.tsx
    │   │       ├── SimpleInput.tsx
    │   │       └── Views/
    │   │           ├── MenuView.tsx
    │   │           ├── LanguageView.tsx
    │   │           ├── CurrencyView.tsx
    │   │           ├── RPCsView.tsx
    │   │           ├── VaultListView.tsx
    │   │           └── MiscSettingsView.tsx
    │   │
    │   ├── 📁 Account/                 # 帳戶相關組件 (33個文件) 👤
    │   │   ├── AccountDeposits.tsx     # 存款顯示
    │   │   ├── AccountDepositsHeader.tsx
    │   │   ├── AccountDepositsTable.tsx
    │   │   ├── AccountDepositsCard.tsx
    │   │   ├── AccountDepositsCards.tsx
    │   │   │
    │   │   ├── AccountDelegations.tsx  # 委託顯示
    │   │   ├── AccountDelegationsHeader.tsx
    │   │   ├── AccountDelegationsTable.tsx
    │   │   ├── AccountDelegationsCard.tsx
    │   │   ├── AccountDelegationsCards.tsx
    │   │   ├── AccountVaultDelegationAmount.tsx
    │   │   ├── AccountVaultDelegationOdds.tsx
    │   │   │
    │   │   ├── AccountWinnings.tsx     # 中獎記錄
    │   │   ├── AccountWinningsHeader.tsx
    │   │   ├── AccountWinningsTable.tsx
    │   │   ├── AccountWinCard.tsx
    │   │   ├── AccountWinCards.tsx
    │   │   ├── AccountWinAmount.tsx
    │   │   ├── AccountWinButtons.tsx
    │   │   │
    │   │   ├── AccountPromotions.tsx   # 促銷活動
    │   │   ├── AccountPromotionsHeader.tsx
    │   │   ├── AccountPromotionsTable.tsx
    │   │   ├── AccountPromotionCard.tsx
    │   │   ├── AccountPromotionCards.tsx
    │   │   ├── AccountPromotionToken.tsx
    │   │   ├── AccountPromotionClaimActions.tsx
    │   │   ├── AccountPromotionClaimableRewards.tsx
    │   │   ├── AccountPromotionClaimedRewards.tsx
    │   │   │
    │   │   ├── AccountVaultBalance.tsx
    │   │   ├── AccountVaultOdds.tsx
    │   │   ├── AccountOdds.tsx
    │   │   ├── CheckPrizesBanner.tsx
    │   │   └── ExternalAccountPageContent.tsx
    │   │
    │   ├── 📁 Vault/                   # 金庫相關組件 (29個文件) 🏦
    │   │   ├── VaultsHeader.tsx        # 列表頁頭部
    │   │   ├── VaultsDisplay.tsx       # 列表展示
    │   │   ├── VaultsTable.tsx         # 表格視圖
    │   │   ├── VaultsDisclaimer.tsx    # 免責聲明
    │   │   ├── VaultCard.tsx           # 單個金庫卡片
    │   │   ├── VaultCards.tsx          # 卡片列表
    │   │   ├── VaultFilters.tsx        # 過濾器
    │   │   ├── VaultListFilterSelect.tsx
    │   │   ├── VaultButtons.tsx        # 操作按鈕
    │   │   │
    │   │   ├── VaultPageContent.tsx    # 詳情頁內容
    │   │   ├── VaultPageHeader.tsx     # 詳情頁頭部
    │   │   ├── VaultPageInfo.tsx       # 基本信息
    │   │   ├── VaultPageCard.tsx       # 信息卡片
    │   │   ├── VaultPageCards.tsx
    │   │   ├── VaultPageButtons.tsx    # 操作按鈕
    │   │   ├── VaultPageStakingContent.tsx
    │   │   ├── VaultPageVaultListWarning.tsx
    │   │   │
    │   │   ├── VaultPagePrizesSection.tsx     # 獎品區塊
    │   │   ├── VaultPageBonusRewardsSection.tsx # 獎勵區塊
    │   │   ├── VaultPageRecentWinnersCard.tsx  # 最近中獎者
    │   │   ├── VaultPageContributionsCard.tsx  # 貢獻度
    │   │   │
    │   │   ├── VaultTotalDeposits.tsx  # 總存款
    │   │   ├── VaultPrizes.tsx         # 獎品
    │   │   ├── VaultPrizeYield.tsx     # 獎品收益率
    │   │   ├── VaultBonusRewards.tsx   # 獎勵
    │   │   ├── VaultWinChance.tsx      # 中獎機會
    │   │   ├── VaultFeePercentage.tsx  # 費用百分比
    │   │   ├── VaultContributions.tsx  # 貢獻統計
    │   │   └── VaultSponsoredDeposits.tsx # 贊助存款
    │   │
    │   └── 📁 Prizes/                  # 獎品相關組件 (6個文件) 🏆
    │       ├── PrizesHeader.tsx
    │       ├── PrizePoolCard.tsx
    │       ├── PrizePoolCards.tsx
    │       ├── PrizePoolDisplay.tsx
    │       ├── PrizePoolPrizesCard.tsx
    │       └── PrizePoolWinners.tsx
    │
    ├── 📁 hooks/                       # 自定義 React Hooks (21個) 🎣
    │   ├── useNetworks.ts              # 網絡相關
    │   ├── useSelectedPrizePool.ts     # 選中的獎池
    │   ├── useSupportedPrizePools.ts   # 支持的獎池
    │   ├── usePrizePoolPPCs.ts         # 獎池 PPC
    │   ├── useRecentWins.ts            # 最近中獎
    │   ├── useDrawsTotalEligiblePrizeAmount.ts
    │   │
    │   ├── useUserTotalBalance.ts      # 用戶總餘額
    │   ├── useUserTotalWinnings.ts     # 用戶總中獎額
    │   ├── useUserTotalDelegations.ts  # 用戶總委託
    │   ├── useUserTotalPromotionRewards.ts
    │   ├── useUserClaimablePromotions.ts
    │   ├── useUserClaimablePoolWidePromotions.ts
    │   ├── useUserClaimedPromotions.ts
    │   ├── useUserClaimedPoolWidePromotions.ts
    │   │
    │   ├── useVaultWinChance.ts        # 金庫中獎機會
    │   ├── useVaultContributions.ts    # 金庫貢獻
    │   ├── useVaultImportedListSrcs.ts # 金庫導入列表
    │   ├── useSortedVaultsByDelegatedAmount.ts
    │   │
    │   ├── useZapTokenOptions.ts       # Zap 代幣選項
    │   ├── useWalletId.ts              # 錢包 ID
    │   └── useSettingsModalView.ts     # 設置彈窗視圖
    │
    ├── 📁 vaultLists/                  # 金庫列表配置 📋
    │   ├── default/                    # 默認金庫列表
    │   │   ├── index.ts
    │   │   ├── mainnet.ts              # 以太坊主網
    │   │   ├── optimism.ts             # Optimism
    │   │   ├── arbitrum.ts             # Arbitrum
    │   │   ├── base.ts                 # Base
    │   │   ├── gnosis.ts               # Gnosis
    │   │   ├── scroll.ts               # Scroll
    │   │   ├── world.ts                # World Chain
    │   │   ├── baseSepolia.ts          # 測試網
    │   │   ├── optimismSepolia.ts
    │   │   ├── arbitrumSepolia.ts
    │   │   ├── scrollSepolia.ts
    │   │   └── gnosisChiado.ts
    │   │
    │   └── meme/                       # Meme 幣金庫列表
    │       ├── index.ts
    │       └── base.ts
    │
    ├── 📁 constants/                   # 常量配置 ⚙️
    │   ├── config.ts                   # 應用配置（RPC、網絡等）
    │   └── theme.ts                    # 主題配置
    │
    ├── 📁 styles/                      # 樣式文件
    │   └── globals.css                 # 全局 CSS（Tailwind 導入）
    │
    └── utils.ts                        # 工具函數                                                                                                      ┌─────────────────────────────────────────────────────┐
│                  apps/app (你的應用)                   │
│  ┌──────────────────────────────────────────────┐  │
│  │         使用所有下面的包和組件                  │  │
│  └──────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
                        ↓↓↓
        
┌──────────────────────────────┐  ┌──────────────────────────────┐
│  📦 packages/                 │  │  🔧 shared/                   │
│  ┌────────────────────────┐  │  │  ┌────────────────────────┐  │
│  │ hyperstructure-        │  │  │  │ react-components       │  │
│  │ react-hooks ⭐         │  │  │  │ (Web3 業務組件)         │  │
│  │ - 170+ React Hooks    │  │  │  └────────────────────────┘  │
│  └────────────────────────┘  │  │  ┌────────────────────────┐  │
│           ↓ 依賴              │  │  │ ui                     │  │
│  ┌────────────────────────┐  │  │  │ (基礎 UI 組件)          │  │
│  │ hyperstructure-        │  │  │  └────────────────────────┘  │
│  │ client-js              │  │  │  ┌────────────────────────┐  │
│  │ - 區塊鏈交互核心        │  │  │  │ generic-react-hooks    │  │
│  └────────────────────────┘  │  │  │ (通用 Hooks)            │  │
└──────────────────────────────┘  │  └────────────────────────┘  │
                                  │  ┌────────────────────────┐  │
                                  │  │ utilities              │  │
                                  │  │ (ABIs + 工具函數)       │  │
                                  │  └────────────────────────┘  │
                                  │  ┌────────────────────────┐  │
                                  │  │ types                  │  │
                                  │  │ (TypeScript 類型)       │  │
                                  │  └────────────────────────┘  │
                                  └──────────────────────────────┘               我让cursor分析了apps/app的结构和packetages，shared文件夹的作用，结果如上，请你帮我分析一下我应该从哪开始阅读？