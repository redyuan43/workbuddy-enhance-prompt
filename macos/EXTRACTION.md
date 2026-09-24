# macOS# macOS 版提取元数据

- **平台时间**：2026-08-27
- **来源文件**：`WorkBuddy.app/Contents/Resources/app.asar` → `main/initialize.js`
- **来源模块**：`packages/workbuddy-server/src/enhance-prompt/handlers.ts`
- **提取方式**：按模板变量名（`DEFAULT_ENHANCE_PROMPT_SYSTEM_TEMPLATE` /
  `DEFAULT_ENHANCE_PROMPT_USER_TEMPLATE`）从反编译产物中原样切出，未做改写。

> **版本标注**：原始提取（commit `06f52a5`）时**未记录客户端具体版本号**，
> 仅确认是 macOS 平台构建（`WorkBuddy.app` 路径即 macOS 专属）。
> 可确知的信息：提取于 2026-08-27；User 模板为"语言一致性"精简版
> （`You are a language consistency assistant` 开头，1248 字节）。
