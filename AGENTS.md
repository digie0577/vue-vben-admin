# AGENTS.md

## 沟通

- 与用户沟通、进度更新、可见的思考摘要和最终回复使用中文；代码、命令、路径、报错保留原文。

## 运行与安装

- 只使用 `pnpm`。`package.json` 有 `preinstall: npx only-allow pnpm`，CI 使用 `pnpm install --frozen-lockfile`。
- 尽量使用 `.node-version` 中与 CI 一致的 Node 版本 (`22.22.0`)。`engines` 允许 `^20.19.0 || ^22.18.0 || ^24.0.0`；`packageManager` 是 `pnpm@10.33.0`。
- 安装后 `postinstall` 会运行 `pnpm -r run stub --if-present`；如果本地 workspace CLI 或配置包解析异常，重新运行 `pnpm install` 或 `pnpm -r run stub --if-present`。
- 依赖大多使用 `catalog:` 或 `workspace:*`；新增依赖时加到最近的 package，再让 `pnpm install` 更新 `pnpm-lock.yaml`。
- 仓库路径及所有父级目录不要包含中文、日文、韩文或空格，否则依赖安装后 Vite 模块解析或启动可能异常。
- 如果 `corepack` 访问 npm 源失败，可设置系统环境变量 `COREPACK_NPM_REGISTRY=https://registry.npmmirror.com` 后再执行 `pnpm install`。

## Monorepo 结构

- Workspace 在 `pnpm-workspace.yaml` 中声明：`apps/*`、`playground`、`packages/**`、`internal/**` 和 `scripts/*`。
- 上游文档涉及 `@vben/web-antd`、`@vben/web-antdv-next`、`@vben/web-ele`、`@vben/web-naive`、`@vben/web-tdesign` 等前端应用；当前工作区实际保留/关注 `@vben/web-antd`，每个应用都维护自己的 adapter、route、locale。
- `playground` 是 package `@vben/playground`，也是唯一带 Playwright e2e 测试的 package。
- `apps/backend-mock` 是 Nitro mock API package。Vite 应用配置会把 `/api` 代理到 `http://localhost:5320/api`。
- `packages/@core/**` 发布底层 `@vben-core/*` 包；`packages/effects/**` 和顶层 `packages/*` 发布 `@vben/*` 共享 UI、stores、request、styles、locales 和 utilities。
- 包依赖方向由 `internal/lint-configs/eslint-config/src/custom-config.ts` 约束：`packages/@core/**` 不能引入 `@vben/*`；`packages/@core/base/**` 不能引入 `@vben/*` 或 `@vben-core/*`；`packages/types`、`packages/utils`、`packages/icons`、`packages/constants`、`packages/styles`、`packages/stores`、`packages/preferences`、`packages/locales` 不能引入 `@vben/*`。
- `internal/*` 放共享 build、lint、tsconfig、vite 工具；`scripts/vsh` 和 `scripts/turbo-run` 提供 root scripts 使用的本地 CLI。

## 命令

- 安装：`pnpm install`。
- 交互式开发选择器：`pnpm dev` 会运行 `turbo-run dev` 并提示选择 package。不要在非交互自动化中使用。
- 非交互开发：当前根脚本提供 `pnpm dev:antd`、`pnpm dev:play`，或使用 `pnpm -F <package-name> dev`。
- 全量构建：`pnpm build`。
- 聚焦构建：当前根脚本提供 `pnpm build:antd`、`pnpm build:play`，或使用 `pnpm -F <package-name> build`。
- 全量类型检查：`pnpm check:type`。聚焦类型检查：对带有该脚本的应用 package 使用 `pnpm -F <package-name> typecheck`。
- CI 等价检查是拆分的：`pnpm test:unit`、`pnpm lint` 和 `pnpm check:type`。PR build workflow 还会运行 `pnpm build`。
- 本地完整仓库检查：`pnpm check` 会运行循环依赖检查、依赖检查、类型检查和 cspell；它不会运行 `pnpm lint` 或单元测试。
- 格式化：`pnpm lint` 只检查；`pnpm format` 会通过 stylelint、oxfmt、oxlint 和 eslint 修复。
- 单元测试：`pnpm test:unit` 使用 `--dom` 和 `happy-dom` 运行 Vitest。运行单个文件使用 `pnpm vitest run <path-to-test> --dom`。
- E2E 测试：`pnpm -F @vben/playground test:e2e` 或 root `pnpm test:e2e`。Playwright 使用 Chromium，`baseURL: http://localhost:5555`，并自行启动 playground server。
- Package export lint：`pnpm publint`。
- 打包体积分析：`pnpm run build:analyze`，用于定位异常依赖体积。
- 预览构建产物优先用 root `pnpm preview`，不要在自动化里引入文档示例中的全局 `live-server`。

## 应用装配

- 在应用 package 中，`#/*` 通过 `package.json#imports` 和 `tsconfig` paths 解析到该应用自己的 `src/*`。
- 应用启动流程是 `src/main.ts` -> `initPreferences()` -> 动态导入 `src/bootstrap.ts` -> component adapter、form adapter、i18n、Pinia stores、access directive、router、Motion，最后 mount。
- `initPreferences()` 的 `namespace` 当前由 `VITE_APP_NAMESPACE`、`VITE_APP_VERSION`、运行环境组合，影响偏好设置、Pinia 持久化和其他本地缓存隔离。
- UI 库相关行为位于各应用自己的 `src/adapter/*` 和 `src/bootstrap.ts` 中的样式导入；修改时更新所有受影响应用，不要假设 adapter 是共享的。
- 公共静态资源若需通过 `src="/xxx.png"` 直接访问，应放到对应应用的 `public/static`，引用路径使用 `/static/xxx.png`。
- Vben Form 组件需要在各应用 `src/adapter/component/index.ts` 中注册并补齐 `ComponentType` 与 `ComponentPropsMap` 类型；Naive UI 表单空值必须用 `null`，不要用 `undefined`，否则重置表单不生效。
- 核心路由在 `src/router/routes/core.ts`。基本路由必须存在且不应修改；需要权限控制的路由从 `src/router/routes/modules/**/*.ts` 自动合并。
- 应用默认只自动合并 `routes/modules/**/*.ts` 动态路由；如需外部路由或静态路由，要打开 `routes/index.ts` 中 `external/static` glob 注释并创建对应目录。外部路由可不使用 Layout，适合被其他系统内嵌，且不显示在菜单中。
- 路由要保持唯一字符串 `name`，尤其是静态/核心路由；没有 `name` 的路由在 `resetStaticRoutes` 时无法按预期删除，`meta.keepAlive` 的懒加载路由也依赖字符串 `name` 重新包装组件名。
- 一级菜单排序由 `meta.order` 控制，并保留 `order: 0` 的语义；二级及以下菜单排序主要按同一父路由 `children` 中的代码顺序控制。
- 多级路由父级 `redirect` 通常可省略，默认指向第一个子路由；自动补 `redirect` 只处理首个子路由 `path` 以 `/` 开头的场景，相对路径不会自动计算完整父级路径。
- 后端/菜单驱动的权限接入使用 `src/router/access.ts`，其中 `pageMap` 是 `import.meta.glob('../views/**/*.vue')`，`layoutMap` 暴露 `BasicLayout`/`IFrameView`。首级自定义菜单不要再配置 `BasicLayout`；有子路由时会移除组件以避免多层 `BasicLayout`。
- `coreRoutes` 不进入权限拦截；无 token 时只有显式声明 `meta.ignoreAccess` 的路由可访问，否则跳登录页并携带 `redirect=encodeURIComponent(to.fullPath)`。
- `route.meta.menuVisibleWithForbidden = true` 会让无权限菜单仍显示，但实际访问会被重定向到 403。
- 单路由多标签页场景用 tab key 控制：优先级为 `query.pageKey` > `meta.fullPathKey !== false` 时的 `route.fullPath` > `route.path`；修改 query 但不希望开新标签时注意 `fullPathKey`。
- `meta.domCached` 可缓存路由 DOM 以减少复杂页面 tab 切换重绘，但会增加内存占用，并且部分 Vue 生命周期不会重新触发；只在明确性能问题时使用。
- 页面若开启路由切换动画，业务页面模板应保持单一根节点；根级注释也算节点，多根节点可能导致页面切换后空白。
- 应用本地翻译从 `src/locales/langs/**/*.json` 加载；第三方组件库 locale 绑定在各应用自己的 `src/locales/index.ts` 中，`dayjs` 未知语言默认回退英语。
- 业务翻译文本不要放进 `@vben/locales`，应放到对应应用的 `src/locales/langs/**`；新增文案时同步补齐所有启用语言包。
- 新增语言包需要同步修改 `packages/locales/langs`、应用 `src/locales/langs`、`packages/constants/src/core.ts` 的 `SUPPORT_LANGUAGES`，以及 `packages/locales/typing.ts` 的 `SupportedLanguagesType`。
- 涉及 `$t()` 的 schema、columns 等长期复用配置优先用函数返回，避免语言切换后数组常量里的文案不更新。
- 偏好设置只覆盖需要变更的配置；修改应用 `src/preferences.ts` 后清空浏览器缓存/本地持久化数据，否则可能继续使用旧配置。
- 扩展项目级业务偏好时使用 `definePreferencesExtension`，并在 `initPreferences({ namespace, overrides, extension })` 传入；扩展偏好也随 namespace 隔离，`resetPreferences()` 会一起重置。
- 扩展偏好字段当前支持 `input`、`number`、`select`、`switch`；`label`、`placeholder`、`tip`、`options[].label` 可直接写 i18n key，`number` 字段的 `min`/`max`/`step` 会在保存和读取旧缓存时校验。
- Pinia 持久化 key 使用 `initStores` 的 `namespace` 作为前缀；多个 app 或环境不要共用同一个 namespace，避免缓存互相污染。

## 权限与请求注意事项

- 前端权限模式下，接口返回的 `userInfo.roles` 必须是数组，并与路由 `meta.authority` 中的权限标识匹配；路由不配置 `authority` 默认可见。
- 按钮级权限可用 `@vben/access` 的 `AccessControl`、`useAccess()` 或 `v-access` 指令；权限码模式需要 `type="code"` 或 `v-access:code`，角色模式使用默认 codes 或 `v-access:role`。
- 如果业务不需要权限码接口，`getAccessCodesApi` 可直接返回空数组，避免无意义接口依赖。
- 对接登录时，默认最小接口约定为：登录接口返回 `accessToken`；用户信息接口至少返回 `roles: string[]` 与 `realName: string`；如果后端响应结构不是 `{ code, data, message }`，优先改应用内 `src/api/request.ts`。
- 实现登出流程时注意防止重复触发导致 `/logout` 死循环，可使用 `isLoggingOut` 标识短路重复登出。
- `@vben/request` 的 `responseReturn` 会影响返回值语义：`raw` 返回原始 `AxiosResponse`，`body` 返回响应 body 且只按 HTTP status 判断，`data` 返回 body 的 `dataField` 并检查业务 `codeField`。
- 应用内 `src/api/request.ts` 的 `defaultResponseInterceptor` 需要按后端格式配置 `codeField`、`dataField`、`successCode`；不要假设所有后端都使用默认 `code=0`、`data` 字段。
- 开启刷新 token 需要同时配置 `preferences.app.enableRefreshToken = true`，并实现 `doRefreshToken` 与 `formatToken`；否则只改接口不会生效。
- 修改认证拦截器时，401 刷新 token 请求必须设置 `config.__isRetryRequest`，并在并发刷新时使用 `refreshTokenQueue` 等待刷新完成；刷新成功或失败都要清空队列，避免无限循环或悬挂请求。
- 通过项目封装的 request 发请求时默认带 `Accept-Language: preferences.app.locale`，服务端若做国际化可直接依赖该请求头。

## 组件注意事项

- 框架组件不是强制使用；若封装不满足需求，可以直接使用原生 UI 组件或自行封装，不要为了复用绕复杂逻辑。
- `Page` 常作为业务页面根容器；当 Modal/Drawer 使用 `appendToMain` 挂载到内容区域时，页面根容器 `Page` 需要开启 `autoContentHeight`，否则弹窗/抽屉高度计算可能不正确。
- Ant Design Vue 适配中表单组件默认使用 `v-model:value`；`Checkbox`、`Radio`、`Switch` 使用 `checked`，`Upload` 使用 `fileList`，新增组件时要配置正确的 model prop。
- 抽屉/弹窗打开后如果表单字段依赖异步数据或插槽渲染，调用 `formApi.setValues()` 前应等待 Vue DOM 更新完成，例如 `await nextTick()`，确保表单字段已挂载。
- `submitOnEnter` 回车提交表单时必须跳过 `textarea`，否则文本域无法换行。
- `VbenForm` 的字段联动必须配置 `dependencies.triggerFields`，否则依赖逻辑不会按预期触发。
- `VbenForm` 中组件展示值与提交值不一致时优先用 schema 的 `valueFormat`；可返回新当前字段值、返回 `undefined` 删除当前字段，或用 `setValue` 拆分成多个字段。
- `VbenForm` 的字段插槽名称等于 `schema.fieldName`，且字段插槽优先级高于 `component` 指定的组件；排查组件不渲染时先看是否存在同名插槽。
- 使用 Zod 表单规则时，`z.string().optional()` 不包含空字符串；允许空字符串需显式 union/or `z.literal('')`。
- Vben Vxe Table 的搜索表单由 Vben Form 接管，不支持 VXE 原生 `formConfig`，使用 `formOptions`；`height: 'auto'` 需要稳定父容器且不要有相邻元素，否则可能高度闪动。
- Vben Vxe Table 的 toolbar slots 由框架固定控制，不按 VXE 原生方式自定义；搜索按钮会合并到 `toolbarConfig.tools`。
- Vben Vxe Table 启用搜索表单后，所有 `form-` 前缀具名插槽会转发给内部 Vben Form。
- 新增 VXE 自定义 renderer 时注意 HMR 重复注册问题，必要时先删除旧 renderer 再注册；固定列中使用 `Popconfirm`/弹层时要处理窄固定列遮挡、body 不跟随滚动和表格层级遮挡之间的冲突。
- 使用 `useVbenModal`/`useVbenDrawer` 的 `connectedComponent` 时，不要把 Modal/Drawer 状态 props 或 slots 传给父连接组件；需要改弹窗属性时通过对应 hook 或 api。
- `VbenModal`/`VbenDrawer` 参数优先级是 `slot` > `props` > `state`；如果已经通过 slot 或 props 控制某属性，`api.setState` 更新同属性不会生效。
- 修改 Modal/Drawer 全局默认行为应在对应应用 `apps/<app>/src/bootstrap.ts` 中调用 `setDefaultModalProps`/`setDefaultDrawerProps`，不要逐个页面重复硬编码。
- Modal/Drawer 提交流程中可用 `api.lock()` 防止重复提交或意外关闭；调用 `close()` 关闭锁定状态弹窗会自动解锁，主动解除用 `unlock()` 或 `lock(false)`。
- `prompt` 自定义组件中 `useAlertContext()` 只能在 `setup` 或函数式组件中调用。
- 通过 `alert`、`confirm`、`prompt` 动态创建的轻量弹窗，在已打开状态下不支持 HMR 热更新；修改相关代码后需关闭并重新打开。
- `ApiComponent` 适合包装 Select、TreeSelect、Cascader 等远程选项组件；多个实例共用同一远程数据源时，建议用 Tanstack Query 包装请求函数以复用并发请求和缓存。
- `ApiComponent.autoSelect` 不应用于多选组件；它只适合当前值为 `undefined` 后自动选择单个选项的场景。
- 修改 `ApiComponent` 请求触发逻辑时，加载中重复触发只标记 pending，当前请求完成并 `nextTick` 后再发起新请求，避免竞态。
- 钉钉扫码登录的 `redirect_uri` 必须是完整 URL。
- `tabbar.refresh(name)` 定向刷新不能传当前路由名称；刷新当前路由时传 router 实例。
- `CountToAnimator` 的 `onFinished`/`onStarted` 事件已废弃，改用 `finished`/`started`。

## 样式、主题与图标

- 当前项目使用 Tailwind CSS v4，不再通过各包的 `tailwind.config.*` 判断是否启用 Tailwind；主题入口在 `internal/tailwind-config/src/theme.css`，Vite 集成在 `internal/vite-config`。
- Vue SFC 中使用 `@apply` 时，`internal/vite-config/src/plugins/tailwind-reference.ts` 会自动注入 `@reference "@vben/tailwind-config/theme"`，一般不要手动重复补。
- 主题 CSS 变量颜色必须使用 HSL 格式，例如 `0 0% 100%`；CSS 变量里不要写 `hsl()` 包裹和逗号。通过 `preferences.ts` 改品牌主色后需清缓存。
- 图标建议统一由 `@vben/icons` 管理：Iconify 图标加到 `packages/icons/src/iconify`，本地 SVG 加到 `packages/icons/src/svg/icons` 并在 `packages/icons/src/svg/index.ts` 导出。

## 内部实现注意事项

- `globalShareState` 是单例，只放与请求无关的全局变量、组件和配置；不要存用户信息等请求态数据，避免后续 SSR 或多请求场景串数据。
- `FormApi.setValues` 的深度合并对普通 object 值支持有限；日期类 `dayjs`/`Date` 已特殊处理，设置复杂对象字段时优先整体替换并验证实际行为。
- `vben-use-form.vue` 和 `vben-form.vue` 避免通过 `extends` 复用 props 类型，会导致热更新卡死；保持当前重复声明方式。
- 连接式 Modal/Drawer 内部扩展 api 时不要直接替换 `reactive` 对象，也不要用 `Object.assign`，否则会丢失响应性或 api 原型方法。
- 修改 Tabbar 持久化逻辑时注意：`Stack` 经 JSON 序列化会退化为普通对象，反序列化后必须重建实例以恢复方法和 getter。
- 不要使用 `getCurrentInstance().ctx` 访问实例上下文；该属性不在公开实例类型中，本地可能能跑但打包后可能异常。
- CJS/UMD 组件在 Vite 下可能解析为 `{ default: Component }`；类似 `JsonViewer` 的导入要保留 `default` 解包逻辑，否则可能出现 `missing template or render`。
- `packages/@core/ui-kit/shadcn-ui/**/*` 是生成代码/内部代码，Tailwind class 顺序检查有意忽略；不要为通过该规则批量改动这些文件。

## Vite 与环境变量注意事项

- 使用 package scripts，例如 `pnpm -F @vben/web-antd dev`，不要直接跑裸 `vite`；`internal/vite-config` 会根据脚本中的 `--mode` 和 package cwd 推导 env 文件。
- 应用 env 加载顺序是 `.env`、`.env.local`、`.env.<mode>`、`.env.<mode>.local`。
- 只有 `VITE_` 前缀变量会暴露到客户端；需要构建后仍可动态修改的变量使用 `VITE_GLOB_*`，并通过 `_app.config.js` 注入。
- 新增 `VITE_GLOB_*` 动态配置时，需要同步补 `packages/types/global.d.ts` 中的 raw/config 类型，并在 `packages/effects/hooks/src/use-app-config.ts` 映射返回字段。
- `useAppConfig(import.meta.env, import.meta.env.PROD)` 只应在应用侧使用；纯包内部不要耦合 `import.meta.env` 或特定构建工具环境变量。
- `VITE_NITRO_MOCK=true` 时，Vite dev 期间只会在端口 `5320` 空闲时启动 `@vben/backend-mock`。
- 生产构建会从 `VITE_GLOB_*` 变量产出 `_app.config.js` 并注入 HTML，全局配置对象会被冻结；`VITE_ARCHIVER=true` 时可能产出 `dist.zip`。
- `VITE_BASE` 作为资源公共路径时需要以 `/` 开头和结尾；部署到非根目录时需同步配置服务端 fallback 路径。
- 自定义全局 loading 时，在应用 `index.html` 同级新增 `loading.html`；必须包含 `id="__app-loading__"` 元素、对应 hidden class，以及 `style[data-app-loading="inject-css"]`。修改 loading 注入逻辑时保留主题缓存读取，避免暗色主题刷新时 loading 闪白。
- Importmap CDN 暂未开启，部分包不支持且网络不稳定；需要启用时优先确认 `esm.sh` 兼容性，`jspm.io` 对 ESM 入口要求更高。

## 构建与部署注意事项

- 删除未用插件、路由或页面后，只要没有被引用就不会进入最终包；体积异常时先用 `pnpm run build:analyze` 定位依赖来源。
- 使用 `history` 路由模式部署时，服务端必须配置 `try_files ... /index.html` 之类的 SPA fallback；否则刷新深层路由会 404。
- 若前端构建了 `.gz`/`.br` 文件，Nginx 还需要启用相应静态压缩模块/配置，例如 `gzip_static`；否则生成压缩文件不一定会被使用。
- Nginx 部署若出现 `.mjs` MIME 类型错误，需要确保 `mime.types` 中包含 `application/javascript js mjs;`。

## Mock 与安全注意事项

- `apps/backend-mock` 的 JWT secret 是演示默认值；复用 mock/auth 逻辑到真实环境前必须替换，并优先通过环境变量或密钥管理注入。
- `apps/backend-mock` 处理 `getQuery(event)` 返回值时要兼容 `string[]`；分页、排序等 query 参数应先规范化，排序字段还应校验为数据项合法 key。
- 新版本不再支持生产环境 mock；生产环境必须使用真实接口。

## 工作流注意事项

- Lefthook pre-commit 会格式化 staged files；删除或新增 workspace package 后，同步清理根 `package.json` scripts、`pnpm-workspace.yaml`、相关 env，并重新执行 `pnpm install`。
- `scripts/vsh/src/index.ts` 比 `scripts/vsh/README.md` 可信；常用 vsh 命令包括 `lint`、`publint`、`check-circular` 和 `check-dep`。
- PR 标题由 `semantic-pull-request` 检查：类型使用 `fix`、`feat`、`docs`、`style`、`refactor`、`perf`、`test`、`build`、`ci`、`chore`、`revert`、`types`、`release` 之一，并且 subject 不要以大写字母开头。
- 删除应用、`playground`、`apps/backend-mock` 等 workspace 内容后，需要同步清理根 `package.json` scripts、相关 env，并重新执行 `pnpm install`；如果删除 mock，还要移除/关闭应用 `.env.development` 的 `VITE_NITRO_MOCK`。
- 删除路由后应同步删除对应页面文件；删除页面/组件前先确认没有路由或其他模块引用，避免构建期动态导入失败。
