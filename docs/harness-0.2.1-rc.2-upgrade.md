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

已执行：`npm install`（1398 包解析无冲突）、`npx patch-package` 全部 24+4 个补丁干净重放、`npm run typecheck` 零错误、`npm run ppt:build`、`npx vitest run` 最终 1964 通过 / 69 失败（升级前基线 1845 项全绿；会话删除、损坏日志、release 契约、plugin-manager 迁移等核心面全部通过）。

未完成（后续工作）：

- **workbench 导航**：`registerSessionStarter` 等桌面接缝未移植到上游新导航架构，`test/workbench-navigation.test.mjs` 等 workbench 系列失败，需按 0.2.1 的 `startSession(options)`/draft 初始化架构重新落位。
- **ppt/office 深度用例**：`ppt:build` 后仍有 7 个文件 26 例失败（`ppt-unbound-mode`、`office-meeting-minutes`、`docx-template-fidelity` 等），多为桌面 PPT 适配器与上游新插槽的对接漂移。
- Windows 验证、安装包打包、真机启动，以及 `plugin-manager` 的 switched/removed 后端回调在 0.2.1 下的真机回归。
