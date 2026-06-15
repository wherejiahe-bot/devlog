# amruta-daily-push logs 2026-06-15 to 2026-06-16

## 2026-06-15
## 2026-06-15

### tithi-monitor GitHub Actions 失败排查修复
- 最新运行失败原因：`FileNotFoundError: [Errno 2] No such file or directory: '/workspace/.tithi_state.json'`
- 邮件其实发送成功了（日志确认：`✓ 邮件已发送: 印度历法提醒：Amavasya`），但保存状态文件时路径不存在导致崩溃
- 修复方案：在 `save_state()` 和 `log()` 函数中增加目录存在性检查，写入前自动创建缺失的父目录
- 邮件文案修改：括号内文字从"（母亲的赐福与向神祇祈求保护…）"改为"（接收母亲的赐福与向神祇祈求保护…）"
- 已推送至 `wherejiahe-bot/tithi-monitor` 仓库

### amruta-actions step2_translate.py 重大修复
- **去重逻辑修复**：删除了 `shutil.copy2` 恢复原始 pairs 的代码。去重后的 pairs 直接写入 pairs.json，不再被原始 pairs 覆盖
- **1:M 关系支持**：新增 `rebuild_pair_html()` 函数，支持 1:1 / M:1 / 1:M 三种关系类型
- **去重时机修正**：去重现在在 HTML 构建之前执行（之前是在 HTML 构建之后才去重，导致邮件用的是去重前的内容）
- **执行结果**：workflow 成功运行 commit 09568d50，archive 已推送，邮件已发送
- **已知问题**：IMA 知识库无匹配时（今天 6.15 文章就全部走阿里云），标题没有单独翻译，留了英文。需要加阿里云翻译标题兜底

### LiteLLM `reasoning_effort` 错误排查
- 错误：`UnsupportedParamsError: openai does not support parameters: ['reasoning_effort'], for model=agnes-2.0-flash`
- 原因：LiteLLM 误将 agnes-2.0-flash 识别为 OpenAI 模型，自动注入了 `reasoning_effort` 参数
- 检查 `models.json`：`drop_params: true` 已经配置，但 WorkBuddy 内部可能绕过了 LiteLLM 直接调用 HTTP
- 搜索了 5 个解决方案，推荐组合使用：方案 1（drop_params 已开启）+ 方案 5（联系官方修复）
- 已向 `wherejiahe-bot/workbuddy-feedback` 创建 issue 供反馈给 WorkBuddy 团队

### 全局记忆新增：模型调用连接稳定性规则
- 写入 `~/.workbuddy/MEMORY.md`：模型调用连接错误时自动重试，不能断掉任务

### Agnes 模型修复：localhost:4000 本地代理未启动问题
- 问题：`models.json` 中 `agnes-2.0-flash` 的 url 指向 `http://localhost:4000/chat/completions`（本地 LiteLLM 代理），但代理服务未运行
- 表现：用户切换 Agnes 模型时各种 HTTP 错误（400/405/500/502）
- 修复：将 url 改回直连官方 API `https://apihub.agnes-ai.com/v1/chat/completions`
- 验证：直连 API 返回 200 ✅，工具调用正常 ✅
- 保留设置：`drop_params: true` + `additional_drop_params: ["reasoning_effort"]` 不变

### Agnes `reasoning_effort` 最终根因修复（2026-06-15 10:20）
- **根因**：`supportsReasoning: true` 让 WorkBuddy 往请求里塞 `reasoning_effort`，但 `useCustomProtocol: true` 绕过 LiteLLM，所以 `drop_params` 和 `additional_drop_params` 完全无效
- **修复**：`supportsReasoning: false` — 从源头阻止 WorkBuddy 发送 `reasoning_effort` 参数
- **教训**：`useCustomProtocol: true` 的模型不走 LiteLLM，`litellm_settings` 对其无效。之前改 `drop_params`/`additional_drop_params`/URL 都是在白费力气

### amruta-actions IMA 搜索策略四层升级（2026-06-15 15:35）
- **关键发现**：source URL 中有正确日期！如 6.15 文章: amruta.today 说 1972 年，但 URL `amruta.org/1979/06/15/...` 显示是 1979 年
- **主人提示**: "阿路的today那个网站上的那个链接上有日期，通过这个日期在IMA知识库里搜索肯定能找到"
- **Commit**: `5e5e76de` (Phase 3) + `0629990d` (Phase 1 用 source URL 日期)
- **最新搜索流程**：
  1. Phase 1 (改): 从 source URL 提取正确日期（如 "1979-06-15"），不再用 amruta.today 的错误日期
  2. Phase 2: 英文正文前两句搜
  3. Phase 3: 月份+日期(去掉年份) + 正文前200字符搜
- **Workflow 运行中**: commit `0629990d`，等待验证 Phase 1 能否找到文章
- **URL 日期格式**：
  - `sahaja.live/1985-06-05-...` → YYYY-MM-DD
  - `sahaja.live/1983-0606-...` → YYYY-MMDD（无分隔符）
  - `amruta.org/1979/06/15/...` → YYYY/MM/DD
  - 用同一个 regex `/(\d{4})[-/](\d{2})[-/]?(\d{2})` 统一提取

### 22:18 - 工作空间外部命令拦截策略
- 主人要求：遇到 `curl` 等系统级命令被拦截弹出"允许/拒绝"框时，**默认选 2（本次会话内始终允许）**，不再询问
- 触发场景：`curl -s --proxy` 访问 GitHub API 等操作时频繁出现这个弹窗，打断工作流
- 理由：主人觉得问"你觉得呢"浪费时间，他想要的是"直接用就行"的流畅体验

### amruta-fix-loop 自动化修复（17:00-17:30）
- **目标**: 检查 amruta-actions workflow 失败并修复
- **根因**: `step2_translate.py` 第 10409 行缩进错误（4-space → 应为 8-space，在 `if not phase1_ok:` 块内）
- **修复**: 用 Python 脚本通过 GitHub API 获取文件 → 修复缩进 → 验证语法 → 提交
- **Commit**: 448d08e641c9
- **验证**: 新 workflow #27563741737 全部通过 ✅（Step1-Step4 均 success）
- **注意**: 旧 log 报 line 20805 错误，实际 Bug 在 line 10409——旧版本文件可能有更多行

## 2026-06-16
### amruta-fix-loop 技能更新（2026-06-16）
- 在 SKILL.md 中加入循环检查 6 月 15 日至 12 日的验证逻辑。
- 新增伪代码描述：在每次 workflow 成功后向前检查前一天，直到 6 月 12 日全部通过后自动暂停 automation（status->PAUSED）。
- 为后续实现提供实现思路和 pause_automation- 将当天所有关于 amruta 每日推送的对话记录追加到 GitHub 仓库的 `DEVELOPMENT-LOG.md`，提交 SHA 185e0d9。
