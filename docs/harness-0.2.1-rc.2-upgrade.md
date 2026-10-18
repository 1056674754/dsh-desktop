# Harness 0.2.1-rc.2 升级记录

仓库内所有 `@deepseek-ai/dsh-*` 依赖从 `0.2.0-rc.2` 固定到 `0.2.1-rc.2`（219 个，`npm view` 逐一核对可用性）。上游在 0.2.1 线删除了两个包，随升级一并移除：

- `@deepseek-ai/dsh-invariants`（止步 0.2.0-rc.2）：ppt-runtime 的 peer 残留声明与测试断言已清理，源代码从未 import。
- `@deepseek-ai/dsh-subagent-in-process-driver`（止步 0.2.1-alpha.1 后并入其他包）：无引用，直接移除。

仓库内插件和 PPT 运行时的 dsh peer 区间追加 `|| ^0.2.1-rc.1`。

## 补丁处理

24 个 dsh 补丁全部迁移到 `+0.2.1-rc.2.patch`（`dsh-client-ui-chat` 补丁删除，见下）。逐包采用三方合并：以 0.2.0-rc.2 原始包 + 旧补丁为参照，合入 0.2.1-rc.2 新源码后用 `patch-package` 重新生成。`@deepseek-ai/dsh` 补丁仍需 `--exclude '^$'` 才包含 package.json。

7 个补丁原样重放（仅改名）：client-file-upload、client-ui-attachment、client-ui-sidebar、llm-deepseek、session-persistence、skill、workspace。

上游 0.2.1 吸收了大量桌面补丁的能力（原生化），迁移时以新版实现为准、仅保留桌面侧仍缺的部分：

- **app-boot**：上游原生提供操作日志、预检拒绝、豁免机制（`dsh plugin allow-version`）。桌面保留 `reportPluginCompatibility`（上报不阻断）与 `readDisabledProfileBundles`；丢弃上游已删除的 `addHarnessSourceSection`。
- **api-session-controller**：会话删除的控制器侧全部保留（`delete` 方法、`handles`/`retainHandle`/`disposeOwned`、typert 注册改写为 0.2.1 的对象映射风格）。上游新增 `claims` 串行化队列与 `composeAgent` 组合 API，与桌面保留项共存。schema 需用 zod 风格（`zod.z.*`）——旧补丁的 schemastery 写法在 0.2.1 bundle 里会拒绝合法输入。
- **api-remotes**：`session/delete` 远程注册改写为 0.2.1 的 map 风格条目（`'session/delete': { service: 'sessionController', ... }`），schema 同样改为 zod。
- **plugin-manager**：上游原生提供 bundle 开关（`selectBundle`）、操作日志、registry plan。桌面保留 `ProfileBundlePackageBackend` 宿主后端接缝（install/remove/switched/removed，market-installer 的 generations 后端依赖它）与两处"兼容性上报不拒绝"。
- **session-persistence-jsonl**：`delete` 实现保留；损坏日志跳过对齐上游新结构并保留 warn 日志。包从默认导出类插件改为 `{ apply, name }` 函数式导出，两个测试已改用新 API。
- **ui-workspace**：上游原生提供 `deleteSession`（含会话删除 UI）。桌面补丁的可见性过滤（isSessionVisible 等）与旧 `registerSessionStarter` 未再保留——workbench 相关测试（`test/workbench-navigation.test.mjs` 等）待按上游新导航架构重新落位。
- **ui-chat**：上游重构了整个聊天视图插槽架构（`conversation.chat.flow`/`chat.node`），桌面补丁的定制（附件打开、错误对话框、footer 插槽）全部被取代，补丁删除。
- **ui-conversation**：保留桌面 `desktop.workbench.frame`、`conversation.hero.modeActions`、`conversation.hero.dock` 插槽（workbench 与 PPT 适配器渲染依赖），同时并入上游新的 `conversation.hero.options`、`conversation.user.text.node`。
- **ui-agent-preset**：桌面的预设导入/导出/搜索/Awesome preset UI 全部保留，叠加上游新的 developerTools 过滤。
- **ui-settings-models**：上游新增推理等级编辑器（保留），桌面图片输入开关与 locale 保留。
- **deliverables**：采用上游新的文件提及匹配（Set 去重 + argumentPath）；桌面"文件夹在文件管理器中打开"特性随旧实现丢弃。

## release 契约

`test/release.test.ts` 的 `HARNESS_VERSION` 更新为 `0.2.1-rc.2`。上游 0.2.1 的官方 bundle 传递引入 session 标题、terminal 会话、会话搜索三个包，从排除清单移除 `dsh-session-title-all-prompts-llm`、`dsh-tool-session-query`、`dsh-tool-terminal`。

## 验证边界

已执行：`npm install`（1398 包解析无冲突）、`npx patch-package` 全部补丁干净重放、`npm run typecheck` 零错误、`npm run ppt:build`、`npm run office:prepare`、`npx vitest run` **2132/2132 全部通过**。

升级过程中额外修复（相对首轮 69 例失败的收敛记录）：

- app-boot/plugin-manager 三方合并遗留的语法错误（游离 return、configure 闭合、`used` 作用域）导致 DSH 入口无法引导，波及 13 个启动类测试。
- 上游 0.2.1 移除 `dsh-skill-office` 的 Python 脚本资产，`check_office.py` 从 0.2.0 tarball 取回并 vendor 到 `scripts/office-runtime/assets-scripts/`，由 prepare.mjs 重新注入。
- pi-ai 1.3.0 目录将 `deepseek-v4-flash` 更名为 `deepseek-v4.1-flash`；`opencode-go` provider 改为 `getModels()` 动态发现。
- `dsh-tool-skill` 0.2.1 新增 `ctx.workingDirectory.ensure` 依赖，测试夹具补 stub。
- `dsh-experimental-schedule-bundle` 在 0.2.1 被移除，generation 投影测试改用安装内的 `dsh-experimental-terminal-bundle`。
- plugin-manager 两处"兼容性上报不阻断"恢复桌面语义文案（`Compatibility is unverified`），bundle 列表补 `switchedOff` 标记。
- ui-workspace 完整移植桌面导航接缝：`registerSessionStarter`、`registerSessionFilter/ReuseFilter/Opener`、`registerWorkspaceFilter/DeleteHandler`、`isSessionVisible/isWorkspaceVisible`、blank 复用过滤，全部挂到上游新的 `connectBlank/createInWorkspace/startSession(options)` 架构上。
- profile-registry 测试对 `npm_config_*` 环境做了隔离（`npx vitest` 注入的用户级 registry 会盖过项目 .npmrc）。

有意放弃、待后续按新架构重做的桌面特性：

- ui-chat 补丁整体（附件打开、文件打开错误对话框、打开消息 footer 卡片）——上游重构了聊天插槽模型；PPT 打开模板卡随 `conversation.chat.userMessageFooter` 插槽一并下线（hero/composer dock 路径保留）。
- deliverables 的"文件夹在文件管理器中打开"与本地路径启发式（上游改为 Set + argumentPath 匹配）。
- 会话树未读角标（`unreadSessionIds`）未随本次移植重新接线。

未执行：Windows 验证、安装包打包与真机启动、市场插件在 0.2.1 下的真机兼容性回归。
