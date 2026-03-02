## Plan: CourseBench Frontend — Next.js + TypeScript + TailwindCSS + MUI 重写

**概要：** 将现有 Vue 2 + Vuetify 2 前端完整迁移到 Next.js App Router + TypeScript + TailwindCSS + MUI (Material Design 2) + SWR + MDX 架构。项目放置在 `frontend-next/` 目录下，与旧前端并存以渐进式迁移。全局状态使用 React Context + `useReducer`，数据获取使用 SWR，策略页和评论均采用 MDX（静态用 `@next/mdx`，动态用 `next-mdx-remote`）。

---

### Steps

**1. 项目初始化**

在仓库根目录创建 `frontend-next/`：

- `npx create-next-app@latest frontend-next --typescript --tailwind --app --eslint`
- 安装核心依赖：
  - `@mui/material @mui/material-nextjs @emotion/react @emotion/cache @emotion/styled` — MUI + Next.js 集成
  - `@mui/icons-material` 或 `@mdi/js` — 图标（保持与旧项目一致用 MDI）
  - `swr` — 数据获取
  - `axios` — HTTP 客户端（保持 API 兼容）
  - `next-mdx-remote` — 动态 MDX 渲染（评论、远程内容）
  - `@next/mdx` + `@mdx-js/react` — 编译时 MDX（策略页面）  
  - `remark-gfm` `rehype-prism-plus` `remark-pangu` — MDX 插件链（GFM 表格、代码高亮、中英间距）
  - `nprogress` — 路由进度条
  - `modern-screenshot` `qrcode` — 截图 + 二维码（按需保留）
  - `js-cookie` — Cookie 操作（替代手写 `document.cookie`）

- 配置 next.config.mjs：
  - MDX 支持：`withMDX` 包裹配置
  - 环境变量：`NEXT_PUBLIC_SERVER_URL`（替代 webpack external `Config.serverUrl`）
  - API 代理：`rewrites()` 将 `/v1/*` 转发到 `http://localhost:5000/v1/*`
  - 构建信息注入：`NEXT_PUBLIC_VERSION`、`NEXT_PUBLIC_BUILD_HASH`、`NEXT_PUBLIC_BUILD_DATE`

- 配置 tailwind.config.ts：
  - 将 Vuetify 的主题色映射到 TailwindCSS custom colors（institute colors, score colors）
  - 保持 MUI 的 breakpoints 与 Tailwind 对齐（xs:0, sm:600, md:900, lg:1200, xl:1536）

**2. MUI 主题 + 布局体系**

创建 src/theme/：

- theme.ts — MUI `createTheme()` 配置：
  - Material Design 2 风格（`@mui/material` 默认即 M2）
  - Palette：primary/secondary colors 映射旧 Vuetify 主题
  - Typography：匹配旧项目字体层级
  - 支持 light/dark 模式切换（旧项目根据 OS preference 自动切换）
  - Shape：`borderRadius: 4`（M2 默认）

- ThemeRegistry.tsx — MUI + Next.js App Router 的 Emotion cache 集成：
  - 使用 `@mui/material-nextjs/v15-appRouter`（或对应版本的 `AppRouterCacheProvider`）
  - 包裹 `ThemeProvider` + `CssBaseline`

**3. App Router 路由结构**

在 `frontend-next/src/app/` 下按现有路由创建文件结构：

```
app/
├── layout.tsx              ← 根布局：ThemeRegistry + Header + Footer + Context Providers
├── page.tsx                ← / → 全部课程（CourseAll）
├── course/
│   └── [id]/
│       └── page.tsx        ← /course/:id → 课程详情
├── teacher/
│   ├── page.tsx            ← /teacher → 全部教师（空占位 → 可跳过或实现）
│   └── [id]/
│       └── page.tsx        ← /teacher/:id → 教师详情
├── user/
│   └── [id]/
│       └── page.tsx        ← /user/:id → 用户主页
├── active/
│   └── page.tsx            ← /active → 邮箱激活
├── reset-password-active/
│   └── page.tsx            ← /reset_password_active → 密码重置确认
├── about/
│   └── page.tsx            ← /about → 关于页面
├── activities/
│   └── page.tsx            ← /activities → 活动页面（bench_reviewer MDX）
├── oauth/
│   └── casdoor/
│       └── page.tsx        ← /oauth/casdoor → Casdoor OAuth 回调
├── recent/
│   └── page.tsx            ← /recent → 最近评论
├── ranking/
│   └── page.tsx            ← /ranking → 赏金排名
├── not-found.tsx           ← 404 页面
├── (policies)/             ← Route group，无 URL 前缀
│   ├── privacy-policy/
│   │   └── page.mdx        ← MDX 静态策略页面
│   ├── terms-of-service/
│   │   └── page.mdx
│   └── comment-policy/
│       └── page.mdx
```

路由兼容说明：旧项目 `/reset_password_active` 使用下划线，Next.js 中保持 `/reset-password-active` 并在 `middleware.ts` 中做 301 重定向兼容旧链接。

**4. 全局状态管理 (Context + useReducer)**

创建 src/contexts/：

- AuthContext.tsx — 替代旧 `global` provide：
  - State: `{ userProfile: UserProfile | null; isLogin: boolean }`
  - Actions: `LOGIN`, `LOGOUT`, `UPDATE_PROFILE`
  - 初始化时从 `preset` cookie 恢复（迁移 useCookie.js 的 base64 encode/decode 逻辑）
  - 导出 `useAuth()` hook

- SearchContext.tsx — 替代旧 `searchInput` provide：
  - State: `{ isRegexp: boolean; keys: string }`
  - 导出 `useSearch()` hook

- SnackbarContext.tsx — 替代旧 `showSnackbar` provide：
  - 使用 MUI `Snackbar` + `Alert`
  - 导出 `useSnackbar()` hook

- CourseFilterContext.tsx — 替代 `savedCourseAllStatus` + `savedCourseAllFilterStatus`：
  - State: `{ page, selected, sortKey, order }`
  - 跨导航持久化

**5. API 层 (SWR + Axios)**

创建 src/lib/：

- api.ts — Axios 实例配置：
  - `baseURL: process.env.NEXT_PUBLIC_SERVER_URL` (替代 `Config.serverUrl`)
  - `withCredentials: true`（Casdoor OAuth 需要）
  - Response interceptor 统一错误处理

- fetcher.ts — SWR fetcher 函数：
  - `const fetcher = (url: string) => api.get(url).then(r => r.data)`

- swr-provider.tsx — SWR 全局配置：
  - `revalidateOnFocus: false`，`revalidateOnMount: true`，`retry: false`（与旧 vue-query 配置对齐）

- API hooks 按模块组织在 src/hooks/：
  - `useCourses()` → `useSWR('/course/all', fetcher)`
  - `useCourse(id)` → `useSWR('/course/' + id, fetcher)`
  - `useTeacher(id)` → `useSWR('/teacher/' + id, fetcher)`
  - `useUser(id)` → `useSWR('/user/profile/' + id, fetcher)`
  - `useCommentsByCourse(id)` → `useSWR('/comment/course/' + id, fetcher)`
  - `useCommentsByUser(id)` → `useSWR('/comment/user/' + id, fetcher)`
  - `useRecentComments()` → `useSWR('/comment/recent', fetcher)`
  - `useRanklist()` → `useSWR('/reward/ranklist', fetcher)`
  - Mutation hooks 使用 `useSWRMutation` 或 plain `axios.post` + `mutate()` 手动 revalidate

**6. TypeScript 类型系统**

创建 src/types/：

- course.ts — 从后端 models/course.go、course_response.go 映射 TypeScript 类型：
  - `Course`, `CourseGroup`, `CourseDetail`, `CourseResponse`
- teacher.ts — 从 models/teacher.go：
  - `Teacher`, `TeacherDetail`
- comment.ts — 从 models/comment.go：
  - `Comment`, `Reply`, `CommentResponse`
- user.ts — 从 models/user.go：
  - `User`, `UserProfile`, `UserResponse`
- common.ts:
  - `ApiResponse<T>`, `PaginatedResponse<T>`, score enums 等

**7. 组件迁移映射 (Vuetify → MUI + TailwindCSS)**

创建 src/components/，按模块组织：

**全局组件** → src/components/layout/：

| 旧组件 | 新组件 | MUI 组件 | 说明 |
|--------|--------|----------|------|
| Header.vue | `Header.tsx` | `AppBar`, `Toolbar`, `IconButton`, `TextField`, `Menu`, `Avatar`, `Dialog` | 拆分：AppBar + SearchBar + UserMenu + LoginDialog + RegisterDialog |
| Footer.vue | `Footer.tsx` | `Container`, `Typography`, `Link` | 版本号从环境变量读取 |
| MenuSideBar.vue | `MobileDrawer.tsx` | `Drawer`, `List`, `ListItem` | 移动端导航 |
| Nothing.vue | `EmptyState.tsx` | `Alert`, `Typography` | 空状态 |

**课程组件** → src/components/course/：

| 旧组件 | 新组件 | MUI 组件 |
|--------|--------|----------|
| `CourseCard.vue` | `CourseCard.tsx` | `Card`, `CardContent`, `Chip`, `Typography` |
| `StatisticCard.vue` | `InstituteFilter.tsx` | `Card`, `Checkbox`, `FormControlLabel` |
| `SelectBar.vue` | `SortBar.tsx` | `Select`, `MenuItem`, `ToggleButton` |
| `ElevatedPagination.vue` | `CoursePagination.tsx` | `Pagination` |
| `DetailCard.vue` | `CourseDetailCard.tsx` | `Card`, `Typography`, `Chip` |
| `FilterBox.vue` | `TeacherGroupFilter.tsx` | `Card`, `Checkbox`, `FormGroup` |
| `CourseCommentCard.vue` | `CommentCard.tsx` | `Card`, `CardContent`, `IconButton` |
| `WritingBox.vue` | `CommentEditor.tsx` | `Dialog`, `TextField`, `Slider`, `Select`, `Switch` |
| `ScoreBoard.vue` | `ScoreBoard.tsx` | `LinearProgress`, custom |
| `ManageCard.vue` | `AdminPanel.tsx` | `Card`, `Button`, `Select` |
| `ReplySection.vue` + `ReplyChainDialog.vue` | `ReplyThread.tsx` | `Dialog`, `List` |

**教师组件** → src/components/teacher/：

| 旧组件 | 新组件 | MUI 组件 |
|--------|--------|----------|
| `Detail.vue` | `TeacherProfile.tsx` | `Card`, `Avatar`, `Typography` |
| `BackgroundImage.vue` | `TeacherBanner.tsx` | Custom div + TailwindCSS gradient |
| `CourseCard.vue` | `TeacherCourseCard.tsx` | `Card` |
| `StatisticCard.vue` | `TeacherStats.tsx` | `Card` |

**用户组件** → src/components/user/：

| 旧组件 | 新组件 | MUI 组件 |
|--------|--------|----------|
| `Login.vue` | `LoginDialog.tsx` | `Dialog`, `Stepper`, `TextField`, `Button` |
| `Register.vue` | `RegisterDialog.tsx` | `Dialog`, `Stepper`, `TextField`, `Checkbox` |
| `Profile.vue` | `UserProfile.tsx` | `Card`, `Avatar`, `Typography` |
| `EditProfile.vue` | `EditProfileDialog.tsx` | `Dialog`, `TextField`, `Button` |
| `CommentCard.vue` | `UserCommentCard.tsx` | `Card` |

**Skeleton Loaders** → src/components/skeleton/：
- 使用 MUI `Skeleton` 组件替代旧 8 个 loader 组件，简化为按页面聚合的骨架屏

**8. MDX 集成策略**

两种 MDX 使用场景：

**a) 静态策略页面 — `@next/mdx` 编译时处理**
- 将 4 个 `.md` 文件移入 `src/app/(policies)/` 目录，重命名为 `.mdx`
- 在 mdx-components.tsx 定义全局组件映射（标题 → MUI Typography，链接 → MUI Link，代码块 → Prism 高亮）
- 每个 MDX 文件可嵌入 MUI 组件（如 `Alert`, `Divider`），实现比纯 markdown 更丰富的排版

**b) 动态评论内容 — `next-mdx-remote` 运行时渲染**
- 创建 src/components/mdx/MdxRenderer.tsx：
  - 接收原始 markdown/mdx 字符串
  - 使用 `serialize()` + `<MDXRemote />` 渲染
  - 插件链：`remark-gfm` (表格/删除线) + `remark-pangu` (中英间距) + `rehype-prism-plus` (代码高亮)
  - 自定义组件映射限制安全范围（评论中禁止 script 等危险标签）
  - 同时也用于 `/activities` 页面的 `bench_reviewer.md` 渲染

**c) MDX 样式**
- 将旧 markdown.css 和 prism.css 迁移为 TailwindCSS 的 `@apply` + `prose` 样式
- 使用 `@tailwindcss/typography` 的 `prose` class 作为基础，覆盖 MUI 主题色

**9. 静态数据 + 常量**

创建 src/constants/：

- institutes.ts — 迁移 `instituteInfo`，添加 TypeScript 类型
- scores.ts — 迁移 `scoreInfo`, `gradingInfo`, `gradingEmojis`
- forms.ts — 迁移 `gradeItems`, `yearItems`, `termItems`, `visibleItems`
- validation.ts — 迁移表单验证规则（`@shanghaitech.edu.cn` 邮箱限制等）

**10. Cookie 认证 + Middleware**

- src/lib/cookies.ts — 迁移 useCookie.js 的 `setPreset`/`getPreset`/`clearPreset` 逻辑，改用 `js-cookie`
- src/middleware.ts：
  - 读取 `preset` cookie 判断登录状态
  - 旧路由重定向：`/reset_password_active` → `/reset-password-active`
  - 屏蔽特殊 teacher ID `100000001`（与旧 migrateRouter.js 一致）

**11. 深色模式**

- MUI 的 `ThemeProvider` 配置 `colorSchemes: { light: true, dark: true }`
- TailwindCSS 配置 `darkMode: 'class'`，与 MUI 模式同步
- `InitColorSchemeScript` 避免 SSR 闪烁
- OS preference 检测逻辑迁移到 `ThemeRegistry.tsx`

**12. 工具函数迁移**

创建 src/utils/：

| 旧 composable | 新 util/hook | 说明 |
|---------------|-------------|------|
| `useParseScore.js` | `parseScore.ts` | 纯函数，`enoughDataThreshold = 3` |
| `useUserName.js` | `getUserDisplayName.ts` | 纯函数 |
| `useTimeUtils.js` | `formatTime.ts` | `unixToReadable()` |
| `useArrayUtils.js` | `arrayUtils.ts` | `sortCmp`, `sumOf`, `averageOf` |
| `useDebounce.js` | `useDebounce.ts` | React hook |
| `useHttpError.js` | `classifyError.ts` | 纯函数 |
| `useExternalUrl.js` | `navigation.ts` | `openInplace()`, `openInNewTab()` |
| `useSponsors.js` | `parseContributorLink.ts` | QQ/GitHub/email 头像解析 |
| `useShare.js` | `shareAssets.ts` | 分享 logo base64 数据 |

**13. NProgress 路由进度条**

- 使用 Next.js App Router 的 `usePathname()` + `useSearchParams()` 监听路由变化
- 创建 src/components/layout/ProgressBar.tsx 包裹 `nprogress`
- 在根 layout 中引入

**14. Docker + 部署**

- Dockerfile — 多阶段构建：
  - Stage 1: `node:20-alpine` 安装依赖 + `next build`
  - Stage 2: `node:20-alpine` 运行 `next start`（或 standalone output）
- 更新 docker-compose.dev.yaml 添加 `frontend-next` 服务
- 生产部署选项：Next.js standalone 模式（自带 Node 服务器）或导出为 static + nginx（若无 SSR 需求）

**15. 推荐迁移顺序**

按页面复杂度渐进迁移，每完成一个可独立测试：

1. **基础框架** — 项目初始化 + MUI 主题 + Layout (Header/Footer) + Auth Context
2. **首页** (`/`) — CourseAll 页面 + CourseCard + InstituteFilter + SortBar + Pagination
3. **课程详情** (`/course/[id]`) — CourseDetail + CommentCard + CommentEditor + ScoreBoard
4. **教师详情** (`/teacher/[id]`) — TeacherDetail + TeacherProfile + TeacherCourseCard
5. **用户页面** (`/user/[id]`) — UserProfile + UserCommentCard + EditProfile
6. **认证流程** — LoginDialog + RegisterDialog + Active + ResetPassword + CasdoorCallback
7. **策略/关于** — About + MDX 策略页 + Activities
8. **管理功能** — Recent + ManageCard (admin) + Ranking
9. **收尾** — 404 页面 + MDX 评论渲染优化 + 深色模式适配 + 移动端适配 + 截图/二维码功能

---

### Verification

- **类型安全**: `tsc --noEmit` 无错误
- **Lint**: `next lint` 通过
- **功能对照**: 逐页面与旧前端 UI 和功能比对，重点检查：
  - 课程搜索（含 regex）正常
  - 评论发布/编辑/删除全流程
  - 登录/注册/OAuth 完整流程
  - 深色模式切换
  - 移动端响应式布局
- **API 兼容**: dev proxy 正常转发 `/v1/*`，所有接口调用返回正确
- **MDX 渲染**: 策略页面、评论 Markdown 正确渲染（代码高亮、表格、中英间距）
- **性能**: Lighthouse 评分 ≥ 90（LCP、CLS、FID）
- **构建**: `next build` 无 warning，bundle size 与旧项目可比

### Decisions

- **App Router** over Pages Router：支持 RSC、nested layouts、streaming，更适合长期维护
- **SWR** over React Query：Vercel 生态更贴合 Next.js，API 更简洁
- **React Context + useReducer** over Zustand：无额外依赖，状态量不大足以胜任
- **next-mdx-remote** for 评论：支持运行时动态序列化，安全可控
- **保持 cookie 认证** over NextAuth：后端 session 机制不变，最小化后端改动
- **TailwindCSS + MUI** 共存：MUI 提供组件骨架，TailwindCSS 处理自定义布局和间距，避免大量 `sx` prop