# Changelog

按版本倒序列出可读变更。机器读取请用 [`updates.json`](./updates.json)；只读哪些文件变动请用 [`manifest.json`](./manifest.json) 的 `last_modified` 字段。

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
