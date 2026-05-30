# Changelog

按版本倒序列出可读变更。机器读取请用 [`updates.json`](./updates.json)；只读哪些文件变动请用 [`manifest.json`](./manifest.json) 的 `last_modified` 字段。

## 1.0.9 — 2026-05-30

**更新微信交流群二维码。**

- `assets/group-qr.jpg` 替换为新的「PPT Skill 交流群-2 群」二维码（6 月 6 日前有效）。
- README 引用路径不变（`./assets/group-qr.jpg`），无需改动正文。

纯资源更新。升级：`python3 scripts/apply_update.py`。

## 1.0.8 — 2026-05-29

**模板推荐改为"3 选 1 + 换一批" + 修正 SKILL.md 的两处文档 bug。**

新规则（SKILL.md 模式 A 第 3 条 / workflow.md A3）：

- 用户已指定模板 → 直接用。
- AI 高度确信唯一最佳 → 可直接用，但先告知选了哪个 + 理由，留否决余地。
- **其余情况（用户没指定 / AI 拿不准）→ 必须让用户选**：用 AskQuestion 给**正好 3 个**候选 + 各自 `preview.png` + 一句话理由；选项永远多带一个**「都不满意，换一批」**，选它就再给**另外 3 个**没出现过的候选，可反复换；用尽仍不满意则追问偏好或转原创。
- 不再允许"不看预览图就凭模糊匹配自作主张定模板"。

顺带修复两处文档 bug（之前版本把本地 SKILL.md 覆盖到公共仓库时引入）：

- 模式 B 不再让 AI 运行未随包发布的 `extract_template.py`，改为 `render_slides.py` + `python-pptx` 现场探查 + explicit address 写 edits.json。
- 目录结构 / 关键脚本表更新为**实际随包发布的 6 个脚本**（build_pptx / render_slides / compute_capacity / check_update / apply_update / build_manifest），删除了 extract_template / stitch_preview / scaffold_detail / batch_extract / enrich_detail 等仅本地数据准备用、未发布的脚本引用。
- "17 套"更正为"19 套"。

纯文档更新，无模板 / 脚本逻辑变动。

升级方式：

```bash
python3 scripts/apply_update.py
```

## 1.0.7 — 2026-05-29

**根治"文字出框" + 强制"同级标题字号一致"。**

问题：生成的 PPT 经常文字太多出框，原因是旧 `max_chars` 是按"原文长度×系数"估的，不是按文本框真实尺寸算的；也没有生成后的出框检查。

改进：

1. **几何精算容量**（新脚本 `compute_capacity.py`，本地数据准备用）：
   - 用文本框真实宽高 + 字号，算出 `chars_per_line`（每行可容字数）、`max_lines`（可容行数）、`max_chars`（容量，含 20% 校准余量，单位为视觉宽度：中文 1 字=1、英文/数字≈0.5）。
   - 解析了 21% 字号继承自母版的占位符（占位符→版式→母版 txStyles），无 None。
   - 退化框（自适应/组合内异形，量不到尺寸）标 `capacity_unknown` 并跳过检测。
   - 全部 19 个模板的 detail.json 已重算；并新增顶层 `type_scale`（字号层级表）+ 每个 slot 的 `level`。
2. **生成时出框检测**（`build_pptx.py`）：
   - 每处改动按视觉宽度对比容量，超了就告警；`--strict` 时直接拒绝保存，必须缩短后重试。
   - autofit（PowerPoint 自动缩字）的框给软提示、不拦截。
   - 经多模板校准：用设计师原文回填时误报率约 0-5%，同时能稳定抓出 1.2 倍以上的真实超长。
3. **编辑铁律新增第 9 条 + 强化第 3 条**：同级标题必须保持模板原字号，**出框只缩短文字、绝不改字号**（改字号会破坏同级一致）。

文件改动：

- 全部 19 个 `templates/*/detail.json`（新增容量字段 + type_scale + level）
- `scripts/build_pptx.py`（出框检测 + `--no-lint` 开关）
- `scripts/compute_capacity.py`（新增，数据准备用，随包附带）
- `SKILL.md` / `references/workflow.md`（编辑规则）
- `VERSION` / `CHANGELOG.md` / `updates.json` / `manifest.json`

升级方式（v1.0.6 → v1.0.7）：

```bash
python3 scripts/apply_update.py     # 拉取重算后的 detail.json + 新 build_pptx.py（纯文本，无 pptx 变动）
```

## 1.0.6 — 2026-05-29

**大幅瘦身：移除嵌入字体 + 把模板移出 Git LFS，根治下载流量超限。**

背景：GitHub 免费 LFS 带宽是 1GB/月（硬上限）。19 个模板的 pptx 在 LFS 里共 ~508MB，其中 **~90% 是原作者保存时嵌入的整套中文字体**（如 top-thesis 77MB 里 76MB 是字体），导致下载人数一多就把月度带宽用爆。

优化：

1. **剥离所有 pptx 的嵌入字体**（`ppt/fonts/*.fntdata` + presentation.xml 的 `embeddedFontLst` + 相关 rels）：
   - 全库 508MB → **63MB**（缩 88%）
   - 版式 / 配色 / 文字位置 / 页数 **零变化**；只是正文改用查看者本机字体（中文系统都有微软雅黑，无感）
   - detail.json / slot 寻址 **完全不变**，编辑流程不受影响
2. **把 pptx 移出 Git LFS，改为普通 git 文件**：
   - 文件已足够小，普通 git 下载**不受 1GB/月 LFS 硬上限约束**
   - 新用户 `apply_update.py`（`git clone --depth 1`）直接拿普通 blob，**LFS 带宽消耗为 0**
3. `apply_update.py`：当远端已不使用 LFS 时自动跳过 `git lfs pull`（纯 git 流程）。

文件改动：

- 全部 19 个 `templates/*/template.pptx`（已瘦身）
- `.gitattributes`（移除 `*.pptx filter=lfs`）
- `scripts/apply_update.py`
- `VERSION` / `CHANGELOG.md` / `updates.json` / `manifest.json`

升级方式（v1.0.5 → v1.0.6）：

```bash
python3 scripts/apply_update.py     # 一次性拉取瘦身后的 19 个 pptx（共 ~63MB 普通 git，不走 LFS）
```

> 注：本月已耗尽的 LFS 带宽要等下个计费周期重置；本版本确保**以后不再依赖 LFS 带宽**。

## 1.0.5 — 2026-05-29

**仓库门面更新：README 重写 + 加入微信交流群二维码。**

变更：

- `README.md`：替换为更直接、更生动的中文介绍（"史上最强原生 PPT Skill / 几大特色 / 兼容 DeepSeek/Mimo/Claude/GPT" 等）。新增"支持定制私有化模板（微信 duge360）"段。
- 新增 `assets/group-qr.jpg`：微信"PPT Skill 交流群"二维码（7 天内有效，过期来 Issues 拿新的）。
- README 加入 `## 交流群` 小节，以居中图片形式展示该二维码。
- 同步 bump `VERSION` / `CHANGELOG.md` / `updates.json` / `manifest.json`，以便已部署在用户机器上的旧版本能通过 `apply_update.py` 拉到新 README + 二维码。

模板 / 脚本无变化。

升级方式（v1.0.4 → v1.0.5）：

```bash
python3 scripts/apply_update.py
```

会下载约 5 个文档 + 1 张二维码图（约 180 KB），无 LFS pptx 流量。

## 1.0.4 — 2026-05-28

**让 AI 真的会自动更新到最新版（修旧版会卡在某个版本的坑）。**

背景：用户反馈说"我用别的 AI 启用这个 Skill 时，它还停在 v1.0.2，没自动更新"。原因是 v1.0.3 之前 SKILL.md 让 AI 先跑 `check_update.py`（只检查不应用），AI 看完输出后**经常忘了**再跑 `apply_update.py`，结果就卡在旧版了。

修复：

- **SKILL.md 顶部加显眼红框**，告诉 AI "第一件事就是跑 `python3 scripts/apply_update.py`"（这一条命令本身就是自检 + 自应用）。
- 把 `check_update.py` 从主路径降级为"只想预览不想升级"的可选工具。
- 明确写出：本 Skill **不会自动 push 更新**，AI 必须主动 pull；如果跳过，会用过期版本。

如何救已经卡在旧版本的 AI：让它跑一次

```bash
python3 scripts/apply_update.py
```

跑完会从 git 增量拉到最新，下次它读 SKILL.md 就会按新规则走。

文件改动：

- `SKILL.md`（顶部新增红框 + 更新机制小节重写）
- `VERSION` / `CHANGELOG.md` / `updates.json` / `manifest.json`

模板 / 脚本无变化，纯 docs。

## 1.0.3 — 2026-05-28

**新增 / 修订 1 条编辑铁律：封面与致谢页按模板能力来，不要硬造。**

背景：v1.0.2 起多数模板已删除自带的"稻壳儿宣传 / 缩略图墙"宣传页，意味着模板里**不一定**有封面（cover）/ 目录（agenda）/ 章节扉页（section_divider）/ 致谢页（ending）。下游 AI 看到 `page_roles` 中某个角色为空时，过去可能会从内容页里挑一张冒充，或者硬塞一段"感谢聆听"。这会破坏视觉一致性。

新规则（SKILL.md > 编辑铁律 第 8 条 / workflow.md > A4）：

- `page_roles.cover == []` → 直接从第一张内容页开始，**不要伪造封面**
- `page_roles.ending == []` → 直接以最后一张内容页收尾，**不要硬造"感谢聆听"**
- `page_roles.agenda == []` → 不强加目录
- `page_roles.section_divider == []` → 不强加分章扉页

简言之："**模板有什么角色就用什么角色**"。

不涉及模板内容变更（pptx / detail.json / preview.png / intro.md 全部保持 1.0.2）。仅更新：

- `SKILL.md`（编辑铁律第 8 条）
- `references/workflow.md`（A4 选页指引）
- `VERSION` / `CHANGELOG.md` / `updates.json` / `manifest.json`

升级方式：

```bash
python3 scripts/check_update.py     # 看到 1.0.2 → 1.0.3
python3 scripts/apply_update.py     # 仅下载 ~6 个文档/元数据文件，无 LFS 流量
```

## 1.0.2 — 2026-05-28

**继续清理模板宣传页 + 拆分超大模板。**

模板裁剪（删除前几页"稻壳儿宣传 + 缩略图墙"）：

| 模板 | 改动 |
|---|---|
| `top-thesis` | 41 → 39（删除前 2 页） |
| `thesis-novice` | 34 → 32（删除前 2 页） |
| `thesis-formula` | 41 → 39（删除前 2 页） |
| `architecture-deck` | 40 → 37（删除前 3 页） |
| `report-savior` | 49 → 44（删除前 5 页） |
| `mckinsey-style` | 42 → 37（删除前 5 页） |

超大合辑拆分：

| 改动 | 详情 |
|---|---|
| 删除 `report-template-massive` | 原 118 页合辑过大、太难选；删除前 5 页宣传后剩 113 页 |
| 新增 `report-massive-models` | **38 页 · 思维模型 + 个人复盘**（原 6-43 页：SWOT/PDCA/KISS/鱼骨/不足/项目复盘） |
| 新增 `report-massive-charts` | **38 页 · 数据图表 + 业绩**（原 44-81 页：齿轮/漏斗/树状/时间轴/财务/销售/季度总结） |
| 新增 `report-massive-reports` | **37 页 · 工作汇报 + 岗位竞聘**（原 82-118 页：核心指标趋势/SWOT/5W1H/金字塔/漏斗逻辑/竞聘） |

每个模板的 `detail.json` 同步：
- 移除被删页面的 slot
- 剩余页面 1..N 重新编号
- 所有 `slot_id` / `title_slot_id` / `body_slot_ids` / `decorative_slot_ids` 更新
- `page_roles` / `skip_pages` / `shape_caution_pages` / `data_charts` slide-number 重映射
- 每个 `intro.md` 的页数描述同步更新
- 每个 `preview.png` 重新拼接 4 张真实内容页

升级方式（v1.0.1 → v1.0.2）：

```bash
python3 scripts/check_update.py     # 列出变动文件
python3 scripts/apply_update.py     # 增量下载（包括新增的 3 个模板 + 删除的 1 个模板）
```

注：`report-template-massive` 在本版本被**完全删除**，旧引用会失效。如果你之前的 edits.json 引用了它，请改用对应的 `report-massive-models` / `report-massive-charts` / `report-massive-reports`（按内容主题对应），并注意 slide_number 已重新计算。

## 1.0.1 — 2026-05-28

**清理 `premium-corp` 模板自带的宣传页 + 加速增量更新流程。**

模板变更：

- `templates/premium-corp/template.pptx`：删除原模板前 3 页（稻壳儿封面 + "35 页精选" 介绍 + 缩略图墙），总页数 38 → 35
- `templates/premium-corp/detail.json`：移除被删页面的 slot 数据，剩余 35 页全部重新编号；slot_id、page_roles、skip_pages、shape_caution_pages、data_charts 等所有 slide-number 引用同步重映射
- `templates/premium-corp/intro.md`：重写"页面结构"小节以反映 35 页真实内容案例
- `templates/premium-corp/preview.png`：重新拼接 2×2 预览图，4 张全部来自真实内容页，不再展示宣传页

脚本改进：

- `scripts/check_update.py`：新增对 `git+` update_source 的支持（之前只支持 HTTP）。现在直接 `git clone --filter=blob:none --no-checkout` 后只拿 `updates.json`，体积小、速度快。
- `scripts/apply_update.py`：克隆时设置 `GIT_LFS_SKIP_SMUDGE=1`，**只对实际变动的 `.pptx` 选择性 `git lfs pull --include=...`**，把更新流量从 ~535MB 降到 ~10MB。
- `scripts/build_manifest.py`：排除 Office 临时锁文件 `~$*.*`。

升级方式（v1.0.0 → v1.0.1）：

```bash
python3 scripts/check_update.py        # 看到 1.0.0 → 1.0.1
python3 scripts/apply_update.py        # 只下载 ~11 个改动文件（含 1 个 LFS pptx）
```

注：v1.0.0 用户首次升级时，本地 `apply_update.py` 仍是旧版（会拉全部 LFS）；从 v1.0.1 起每次升级仅下载真正变动的文件。

## 1.0.0 — 2026-05-27

**首次发布。**

新增：

- 核心脚本：`render_slides.py` / `extract_template.py` / `stitch_preview.py` / `build_pptx.py` / `batch_extract.py` / `scaffold_detail.py` / `check_update.py` / `apply_update.py`
- 17 个内置模板（每个 = `template.pptx` + `intro.md` + `detail.json` + `preview.png`）：
  - `minimal-business-summary` — 简约商务总结汇报（深蓝白，16 页）
  - `red-patriot-youth` — 新时代新青年红色教育（党政红，16 页）
  - `cute-orange-class` — 橙色可爱卡通教学（暖橙，17 页）
  - `quarterly-illust` — 蓝灰酸性插画季度总结（Y2K 亮蓝，19 页）
  - `geometric-summary` — 多彩几何工作总结（4 主色，21 页）
  - `red-patriot-general` — 红色爱国主题教育通用（党政红，25 页）
  - `thesis-novice` — 多专业开题方法论库（墨绿，34 页）
  - `premium-corp` — 高级感大厂 PPT 合辑（酱红，38 页）
  - `architecture-deck` — 领导爱的架构图合辑（深蓝，40 页）
  - `thesis-formula` — 开题报告万能公式（暖米色，41 页）
  - `data-viz-deck` — 数据可视化合辑（深蓝砖红，41 页，含原生 chart）
  - `top-thesis` — 名校开题报告合辑（酒红，41 页）
  - `mckinsey-style` — 麦肯锡风专业模板（酱红，42 页）
  - `report-savior` — 汇报救命合辑（深蓝，49 页）
  - `operations-deck` — 运营 PPT 合辑（深蓝，52 页）
  - `competition-speech` — 竞聘述职合辑（深蓝砖红，59 页）
  - `report-template-massive` — 118 页超大工作汇报合辑（深蓝）
- 参考文档 `references/` 系列
- 版本机制：`VERSION` / `CHANGELOG.md` / `updates.json` / `manifest.json`
- 字体 fallback 链：WenQuanYi Micro Hei → DengXian → Noto Sans SC → PingFang SC
- 非商业使用声明在 SKILL.md 顶部明确化
