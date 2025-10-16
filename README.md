# 快速上手指南：20 篇文案风格总化系统

本项目包含 `IMPLEMENTATION_BLUEPRINT.md`，用于详细描述整套多篇文章风格分析与提示词生成流水线。本指南针对“如何实际执行项目”给出概览、步骤与常见问题，帮助你快速落地。

## 目录结构
- `IMPLEMENTATION_BLUEPRINT.md`：完整实施蓝图，包含架构设计、并发策略、算法细节及验收清单。
- `data/`（需自行创建）：输入、输出与临时文件建议路径。
  - `data/input/`：放置 20 篇 `.txt` 原文。
  - `data/out/`：存放每篇分析产物（`*.article.json`、`*.prompt.json`）。
  - `data/tmp/`：中间汇总文件（例如 `prompts_20x.json`）。
  - `data/final/`：终极交付（`StylePrompt.json` 等）。
- `apps/server/`：Node.js/Express 后端（按蓝图创建）。
- `apps/web/`：Next.js 前端（按蓝图创建）。
- `scripts/analysis/`：Python 分析脚本集合。

> 若以上目录尚未建立，可按蓝图中的说明在初始化阶段创建。

## 快速执行流程
1. **准备环境**
   - 安装 Python 3.10+、Node.js 18+、PowerShell 7（Windows 10 环境）。
   - 设置 deepseek-reasoner API Key 为 `DSK_API_KEY` 环境变量。
   - 参考蓝图 §10“Win10 运行”执行虚拟环境、依赖安装命令。

2. **放置输入文件**
   - 将待分析的 20 篇 `.txt` 文档放入 `data/input/` 目录。
   - 文件名建议仅包含英文字母、数字与连字符，避免空格。

3. **启动服务**
   - 后端（`apps/server`）运行 `node index.js`，提供上传、状态与导出 API。
   - 前端（`apps/web`）运行 `npm run dev`，用于文件上传与结果展示。
   - 后端配置 `MAX_CONCURRENCY`（缺省为文章数）以控制 Python 子进程并发度。

4. **触发分析**
   - 通过前端上传文件或调用 `POST /api/upload` 接口，将文章写入 `data/input/`。
   - 调用 `POST /api/analyze` 或在上传后自动触发：
     - 后端按并发限制执行 `scripts/analysis/run_article.py`。
     - 每篇输出 `data/out/<name>.article.json` 与 `<name>.prompt.json`。
   - 全部完成后，后端调用 `scripts/analysis/merge_prompts.py` 聚合为 `prompts_20x.json`。
   - 后端使用 `llm_style_unify.py` 或内置 LLM 调用流程生成 `Style_Prompt_20x.json` 与 `StylePrompt.json`。

5. **查看与导出结果**
   - `GET /api/result?jobId=`：查询每篇分析、提示词及汇总文件的索引。
   - `GET /api/export/style-prompt`：导出最终 `StylePrompt.json`。
   - 前端页面展示三层结果（篇级分析、20x 总化、终极 Style Prompt）。

## 常见问题（FAQ）
- **是否必须一次 20 篇？** 蓝图允许任意篇数；并发度默认等于输入篇数，可通过 `MAX_CONCURRENCY` 调整。
- **脚本失败怎么办？** 后端会自动重试 3 次，并在 `data/out/` 生成 `.error.json` 供排查。
- **如何验证结果？** 参照蓝图 §11 的验收清单，重点关注 JSON 字段完整性、共性统计与风格复刻度。
- **能否只用脚本不启服务？** 可以在 PowerShell 中手动运行蓝图 §10 提供的并发示例命令与后续汇总流程。

## 深入阅读
欲了解算法细节（如 TF-IDF、Weirdness、Top-K 精彩点评分）以及 LLM Prompt 模板，请参见 `IMPLEMENTATION_BLUEPRINT.md` 对应章节（§5–§8）。该蓝图也是团队协作与审核的唯一规范文档。

如在执行过程中遇到问题，可从以下角度排查：
- 检查环境变量与依赖是否齐全。
- 查看后端日志（pino 输出）确认任务状态。
- 检查 `data/out/` 是否生成预期 JSON，必要时手动运行单篇脚本验证。

祝执行顺利！
