# 实施蓝图：20 篇文案风格总化系统

本文档提供在 Windows 10 环境下无需 Docker 即可执行的端到端开发说明，供工程团队（含 Codex/自动化代理）直接按图实施。整体遵循 deepseek-reasoner 统一模型、篇数即并发度的要求，并覆盖前后端、分析脚本与 LLM 调用的全链路设计。

---

## 1. 系统概览与约束

- **目标**：对 20 篇输入文案执行客观统计 + LLM 分析，生成每篇 `ArticleAnalysis`、`ArticlePrompt`，再经多轮总化输出全局 `StylePrompt.json`。
- **统一模型**：全部 LLM 交互封装为 `llmInvoke(prompt, params)`，默认 `temperature∈[0.6,0.8]`、`top_p=0.9`，且只返回结论。
- **并发策略**：默认并发 = 输入 TXT 数，允许通过 `MAX_CONCURRENCY` 上限控制。
- **部署限制**：禁止 Docker，需保证 PowerShell + Node.js + Python 的 Win10 可落地执行。
- **产物链路**：
  1. `ArticleAnalysis`（篇级详细分析）
  2. `ArticlePrompt`（篇级提示词）
  3. `prompts_20x.json`（20 篇合并提示词）
  4. `Style_Prompt_20x.json`（LLM 总化）
  5. `StylePrompt.json`（终极总分析，可迁移至任意大模型）

---

## 2. 目录与子工程

```
text-fix/
├─ apps/
│  ├─ server/        # Node/Express 后端
│  └─ web/           # Next.js 前端
├─ scripts/
│  └─ analysis/      # Python 分析脚本
├─ data/
│  ├─ input/         # 原始 txt 文档
│  ├─ out/           # 篇级分析输出
│  └─ final/         # StylePrompt & 导出物
├─ logs/
├─ .env.example
└─ IMPLEMENTATION_BLUEPRINT.md
```

---

## 3. 技术栈与依赖

### 3.1 后端（apps/server）

- **语言**：Node.js + TypeScript
- **关键依赖**：`express`, `multer`, `zod`, `dotenv`, `pino`, `uuid`, `cross-spawn`, `p-limit`
- **职责**：
  - 接收多文件上传、生成 `jobId`
  - 调度 Python 分析脚本并管理并发/重试
  - 聚合 `ArticlePrompt` → `prompts_20x.json`
  - 调用 `llmInvoke` 生成 `Style_Prompt_20x.json`、`StylePrompt.json`
  - 提供导出接口（JSON/MD/PDF）

### 3.2 前端（apps/web）

- **技术栈**：Next.js 14(App Router)、Tailwind CSS、`@tanstack/react-query`、`shadcn/ui`、`recharts/echarts`
- **功能**：
  - 上传面板与进度轮询
  - 展示篇级分析（三层结果）
  - 横向共性对比图表
  - 提示词工坊（允许微调结构、语气并导出）

### 3.3 Python 分析脚本（scripts/analysis）

- **依赖**：`jieba` 或 `pkuseg`, `regex`, `nltk`, `scikit-learn`, `numpy`, `pandas`, `networkx`, `requests`
- **可选**：`spacy`(zh)、`fastapi`
- **职责**：文本清洗、切分、统计、客观评分、候选术语抽取、调用 LLM，输出 JSON。

### 3.4 LLM SDK 封装

- `llmInvoke(prompt, params)`：对 deepseek-reasoner 的统一封装，支持温度、top_p、重试与响应 JSON 校验。

---

## 4. 端到端流程

1. 前端上传 20 篇 TXT → `POST /api/upload`
2. 后端保存于 `data/input/`，返回 `jobId`
3. `POST /api/analyze` 触发并发任务：`run_article.py`
4. 每篇输出 `data/out/<name>.article.json` 与 `.prompt.json`
5. 所有成功后执行 `merge_prompts.py` → `prompts_20x.json`
6. 调 LLM：
   - 总化：`prompts_20x.json` → `Style_Prompt_20x.json`
   - 终极：`Style_Prompt_20x.json` → `StylePrompt.json`
7. `GET /api/result?jobId=` 返回产物索引；前端展示与导出。

---

## 5. 后端 API 设计

| 方法 | 路径 | 功能 |
|------|------|------|
| POST | `/api/upload` | 多文件上传，返回 `jobId` |
| GET  | `/api/status?jobId=` | 查询队列/运行/成功/失败状态 |
| POST | `/api/analyze` | 根据 `jobId` 启动并发分析 |
| GET  | `/api/result?jobId=` | 返回篇级与全局产物路径 |
| GET  | `/api/export/:type` | 导出 `style-prompt.json` / pdf / md |

**并发调度示例**：

```ts
import pLimit from 'p-limit'
const files = listInputTxt(jobId)
const limit = pLimit(Number(process.env.MAX_CONCURRENCY) || files.length)
await Promise.all(
  files.map(file => limit(() => spawnPython('run_article.py', { input: file })))
)
```

---

## 6. Python 分析流程

### 6.1 `run_article.py`

- **调用方式**：`python run_article.py --input <path> --out data/out/<name>`
- **核心步骤**：
  1. `clean_text()`：去 HTML/emoji/URL/广告尾巴、繁转简、标点统一、>80% 重复段落去重
  2. `sent_split()` / `para_split()` 切句、切段
  3. 统计：词频、TF-IDF、Weirdness、句长、人称/副词/祈使占比、句式模板命中（设问/对比/排比/转折等）
  4. 客观评分：
     - `RareScore = 1 - tfidf_norm` 或 `weirdness`
     - `PositionScore`：开头 10%/转折窗口加权
     - `RhetoricPeak`：修辞模板数 + 情感幅度Δ + 语义突变Δ
     - `Obj = 0.4*Rare + 0.3*Position + 0.3*Peak`，取 Top-K=5
  5. LLM 宏观分析（Prompt A）
  6. LLM 微观分析（Prompt B）
  7. 术语甄别（Prompt C）
  8. 组装 `ArticleAnalysis`
  9. LLM 生成 `ArticlePrompt`（Prompt D），含 7 段式原文示例
 10. 输出 `.article.json` 与 `.prompt.json`

### 6.2 `merge_prompts.py`

1. 读取全部 `.prompt.json`
2. 结构、推进、钩子、证明等维度按频率排序，保留主/次组合
3. 合并并去重 `signature_words`、`seven_paragraphs`
4. 输出 `prompts_20x.json`

### 6.3 `llm_style_unify.py`

1. 调用 LLM（Prompt E）生成 `Style_Prompt_20x.json`
2. 再次调用 LLM（Prompt F）生成终极 `StylePrompt.json`，要求结构/推进 MUST，并补足 7 段示例与通用提示词模板。

---

## 7. 非 AI 算法细节

- **Weirdness**：`(freq_doc/len_doc) / (freq_bg/len_bg)`，背景语料可选新闻/百科
- **TF-IDF**：对 2~4 gram 取平均最大值
- **PositionScore**：开头 10% = 1.0，中部 = 0.7，转折窗口 = 0.9
- **RhetoricPeak**：修辞命中数 + 情感幅度Δ + 语义距离Δ
- **综合得分**：`0.4*Rare + 0.3*Position + 0.3*Peak`
- **语言特征统计**：通过词典 + 正则获得副词、祈使、人称等占比（单位：每千词或占比）。

---

## 8. 数据契约

- `ArticleAnalysis`：字段遵循主方案 §4.1（宏观、微观、超微观、统计等，全量输出并经 `zod` 校验）。
- `ArticlePrompt`：结构详见 §7.2，含 `style_card` 与 `seven_paragraphs`。
- `StylePrompt`：包含风格卡、通用提示词、7 段示例、适配片段、负面清单，结构与推进为 MUST。

---

## 9. LLM Prompt 模板

A. **单篇宏观**：参见 §8.A，强调仅返回 JSON。

B. **单篇微观**：参见 §8.B，输出 `blocks`, `progression_blueprint`, `hook`, `closing`, `evidence_methods`, `structure_highlight`。

C. **术语甄别**：参见 §8.C，输出 `metaphor_terms`、`domain_terms`。

D. **篇级提示词生成**：参见 §8.D，确保结构/推进优先、示例源自原文。

E. **20x 合并**：参见 §8.E，产出 `Style_Prompt_20x.json`。

F. **终极总分析**：参见 §8.F，产出 `StylePrompt.json`，结构/推进 MUST。

---

## 10. 错误处理与重试

- Python 脚本异常 → 输出 `.error.json`，后端最多重试 3 次（指数退避）。
- LLM 超时/限流 → 退避 + 重试；若仍失败记录 `missing_fields` 并触发补采。
- JSON 解析失败 → 使用“格式纠正” Prompt 再次调用 LLM。

---

## 11. Windows 10 部署指南

```powershell
# Python 环境
y -3 -m venv .venv
. .\.venv\Scripts\Activate.ps1
pip install -r scripts\analysis\requirements.txt

# 前后端安装
cd apps\web; npm install; cd ..\server; npm install; cd ..\..
```

```powershell
# 启动
cd apps\server; node index.js
cd ..\web; npm run dev
```

```powershell
# 并发 = TXT 数示例
$files = Get-ChildItem .\data\input\*.txt
$max = $files.Count
$files | ForEach-Object -Parallel {
  python .\scripts\analysis\run_article.py --input $_.FullName --out .\data\out\$($_.BaseName)
} -ThrottleLimit $max
```

```powershell
# 合并与总化
python .\scripts\analysis\merge_prompts.py --in .\data\out --out .\data\final\prompts_20x.json
```

环境变量（`.env`）：

```
DSK_API_KEY=你的deepseek-reasoner密钥
MAX_CONCURRENCY=auto
OUTPUT_DIR=./data/final
```

---

## 12. 验收清单

1. 每篇 `ArticleAnalysis` 字段齐全并通过 `zod` 校验。
2. `prompts_20x.json` 含主/次结构、推进、钩子、证明、语气、风格、文字性格共性。
3. `StylePrompt.json` 包含风格卡、通用提示词、7 段示例、适配片段、负面清单，且结构/推进标注 MUST。
4. 样本外 5 篇文案复刻度 ≥ 0.90（风格向量相似度均值）。
5. 20 篇输入触发 20 并发或受 `MAX_CONCURRENCY` 限制。
6. 故障注入（随机两篇失败）→ 自动重试后仍产出完整结果。

---

> 完成本蓝图后，可直接按模块实现代码、脚本与界面；所有 prompt 模板与接口定义已对齐交付物要求。

