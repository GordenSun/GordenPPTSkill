# PPT Builder Skill

> **版本** 见同目录 [`VERSION`](./VERSION)；每次启用本 Skill 必须先执行"更新检查"环节。
>
> **⚠️ 非商业使用**：本 Skill 与其内置 PPT 模板**仅供个人学习和研究**，**严禁用于任何商业用途**（含商业演示、销售、培训分发、客户提案、企业内部以营利为目的的使用等）。模板素材来自第三方设计师作品，二次商用需要获取原作者授权。

## 何时使用本 Skill

当用户的需求满足下列任何一项就调用本 Skill：

- 需要"做一份 PPT / 演示文稿 / 幻灯片"，无论是工作汇报、年终总结、季度复盘、项目提案、述职竞聘、教学课件、开题答辩、读书报告……
- 用户给了文字大纲或一段描述，希望"做成 PPT"
- 用户拿来一份 .pptx 文件，希望"按这个模板做一份新的"
- 用户希望"调整 / 编辑 PPT 的某些文字"，且要求"不破坏排版"
- 用户希望对比多个 PPT 模板，选一个合适的

## 一切开始之前 —— 更新自检

**每次启用本 Skill 必须先执行这一步**，否则你可能用的是过期版本：

```bash
python3 scripts/check_update.py
```

该脚本会：
1. 读取本地 `VERSION` 和 `updates.json`
2. 对比远端 `updates.json`（如果配置了 `update_source`）
3. 输出三类信息：
   - "Up to date" → 直接继续
   - "Need update: vX → vY, changed files: [...]" → 按提示运行 `apply_update.py`，只下载变动 / 新增的文件
   - "Update source unset" → 跳过

后续每次会话只在第一次执行时检查；同一会话内不重复检查。

## 三种模式

收到用户需求后，先判断走哪种模式：

### 模式 A：从内置模板里挑

**默认走这条路。** 17 套内置模板覆盖了绝大多数中文场景。

1. **读 [`templates/INDEX.md`](./templates/INDEX.md)** ——一份精简清单，列出每套模板的风格、主色、适用场景、页数。
2. **匹配用户输入** —— 把用户描述（场景、风格关键词、所需页面类型、颜色偏好）和每个模板的 intro.md 对比，挑出 1-3 个最匹配的。
3. **不确定时给用户看预览图** —— 把候选模板的 `templates/<slug>/preview.png` 通过 AskQuestion 一起发给用户，让用户选。
4. **拿到目标模板后**：
   - 读 `templates/<slug>/intro.md`（高度浓缩，告诉你这个模板的特性）
   - 读 `templates/<slug>/detail.json`（结构化数据，告诉你每页 / 每个文本位的细节）
   - 按 [模式 A 工作流](./references/workflow.md#mode-a) 选页、写 `edits.json`、跑 `build_pptx.py`

### 模式 B：用户自己带 PPT 模板

当用户提供了 .pptx 文件且明确希望以它作模板时：

1. 把用户的 pptx 当作"未知模板"
2. 用 `scripts/render_slides.py` + `scripts/extract_template.py` 解析它，**得到 raw.json 和每页 PNG**
3. 自己看每页（PNG + raw.json 文本）：
   - 推断每页是什么角色（封面 / 目录 / 章节扉页 / 内容页 / 结束 / 模板宣传）
   - 推断每页适合放什么内容
   - 标记不要使用的页面（模板宣传 / "稻壳儿" 之类）
4. 按 [模式 B 工作流](./references/custom-template-workflow.md) 进行后续选页 + 文字替换
5. **不要修改用户模板原文件**；所有改动写到新的 output.pptx

### 模式 C：完全原创（不基于任何模板）

当用户明确要求"原创设计 / 不用模板 / 简单干净的样式"时：

1. **创建尽量简洁的版式** —— 大量留白、对齐严格、装饰极少
2. 单页元素 ≤ 4 个，避免复杂图形 / 图标群组
3. 字体：英文 Arial / Helvetica；中文 微软雅黑 / 思源黑体；标题加粗即可
4. 主色 1 个 + 灰阶 + 白底；不要拼凑多种风格
5. 用 `python-pptx` 直接代码生成，参考 [`references/original-design-guide.md`](./references/original-design-guide.md)

## 编辑铁律（所有模式通用）

1. **不改排版** —— 只改文字。形状的位置、大小、颜色、字体、字号、行距，都不动。
2. **所有占位文字必须替换** —— 模板里的 "Question 1" / "Vivamus..." / "Key Words Here" / "项目名称" 等占位文本都必须用真实内容替换。一份完成的 PPT 里不应出现任何示例占位词。
3. **字数受 `max_chars` 约束** —— detail.json 的每个 slot 都标注了最大字符数；超过会换行/截断破坏版面。
4. **数字 / 序号默认不动** —— `editable: false` 的 slot 是装饰性 "01/02/1/2/%" 之类，除非用户明确要求改顺序，否则保持。
5. **图形 / 图表通常无法同步** —— 装饰性的进度条 / 圆环 / 旗帜路径 / 流程箭头是固定形状；改了百分比文字不会改弧长。每个模板 detail.json 里如有这类页面会在 `cautions` 字段列出。
6. **真实数据图表才能改数据** —— 如果某页有 PPT 原生 chart（`shape.has_chart=True`），可以用 `build_pptx.py --chart-data` 同步更新；详见 [`references/chart-editing.md`](./references/chart-editing.md)。
7. **章节名前后呼应** —— 改了目录章节名，对应的分章扉页 + 内容页面包屑文字都要同步改。
8. **封面 / 致谢页按模板能力来，不要硬造** ——
   - 读 `detail.json` 的 `page_roles`，如果 `cover` 数组为空，**直接从第一张内容页开始**，不要拿一张内容页当封面用，更不要从其他模板临时凑一张封面页。
   - 如果 `ending` 数组为空，**直接以最后一张内容页收尾**，不要硬造"感谢聆听"。
   - 同理，`agenda` 空 → 不强加目录；`section_divider` 空 → 不强加分章扉页。
   - 也就是说：**模板有什么角色就用什么角色**，少一个角色就少一页，不要破坏视觉一致性去拼凑。这条规则在 v1.0.3 起对所有模板生效。

## 标准工作流（模式 A）

```bash
# 1. 从 templates/INDEX.md 选定一个模板，例如 minimal-business-summary
TEMPLATE=templates/minimal-business-summary

# 2. 读两个文件
#    - $TEMPLATE/intro.md     -> 模板风格 / 适用场景 / 结构概述
#    - $TEMPLATE/detail.json  -> 每页 / 每个文本位的详细描述

# 3. 自己决定要用哪些页（用 detail.json 的 page.role 和 use_for）
#    生成 edits.json：
#    {
#      "template_slug": "minimal-business-summary",
#      "selected_slides": [1, 2, 3, 5, 7, 9, 10, 12, 13, 14, 16],
#      "edits": [
#        {"slide": 1, "slot_id": "cover_title_cn", "new_text": "2026 年度复盘"},
#        ...
#      ]
#    }

# 4. 跑构建
python3 scripts/build_pptx.py \
    $TEMPLATE/template.pptx \
    edits.json \
    out/final.pptx \
    --detail $TEMPLATE/detail.json \
    --strict

# 5. （可选）生成最终预览图给用户看
python3 scripts/render_slides.py out/final.pptx out/renders
python3 scripts/stitch_preview.py out/renders out/preview.png --pages 1 3 6 9
```

## 目录结构

```
ppt-skill/
├── SKILL.md               ← 本文件
├── VERSION                ← 当前版本号（例 1.0.0）
├── CHANGELOG.md           ← 人类可读变更日志
├── updates.json           ← 机器可读版本增量索引
├── manifest.json          ← 所有受版本管理文件的 sha256 与版本归属
├── README.md              ← 仓库概览（用户阅读）
├── scripts/
│   ├── render_slides.py       # pptx → PDF → PNG
│   ├── extract_template.py    # pptx → raw.json 结构化文本+形状
│   ├── stitch_preview.py      # 4 页拼接为 preview.png
│   ├── build_pptx.py          # 按 edits.json 选页+换字
│   ├── batch_extract.py       # 批处理
│   ├── scaffold_detail.py     # 由 raw.json 自动生成 detail.json 草稿
│   ├── check_update.py        # 检查远端是否有更新
│   └── apply_update.py        # 增量更新本地文件
├── references/
│   ├── workflow.md
│   ├── pptx-edit-schema.md
│   ├── custom-template-workflow.md
│   ├── chart-editing.md
│   └── original-design-guide.md
└── templates/
    ├── INDEX.md
    └── <slug>/
        ├── template.pptx
        ├── intro.md       # 高度浓缩简介
        ├── detail.json    # 详细页面 / slot 数据
        └── preview.png    # 4 页 2×2 拼接预览图
```

## 关键脚本一句话说明

| 脚本 | 干嘛用 |
|---|---|
| `render_slides.py` | 把任意 pptx 渲染成每页一张 PNG（用 LibreOffice + pdftoppm） |
| `extract_template.py` | 把 pptx 的形状 / 文本结构 dump 成 raw.json，并产出 text_overview.md |
| `stitch_preview.py` | 选 4 张 PNG 拼成一张 2×2 预览图 |
| `build_pptx.py` | 按 `edits.json`（含选页、文字替换）从模板生成最终 pptx |
| `scaffold_detail.py` | 由 raw.json 生成 detail.json 初稿（机械填充 slot_id 等） |
| `check_update.py` | 对比本地 VERSION 和远端 updates.json，告诉你要不要更新 |
| `apply_update.py` | 按 updates.json 的 delta 列表只下载变动文件 |

## 字体说明

模板 XML 里大量使用 `微软雅黑`。如果运行环境没有该字体，配合 `~/.config/fontconfig/fonts.conf` 把它别名到本地已安装的字体（推荐顺序：WenQuanYi Micro Hei → DengXian → Noto Sans SC → PingFang SC）。预览图正是用这条 fallback 链渲染的。

最终交给用户的 .pptx 在 PowerPoint / WPS / Keynote 里打开时会自动用宿主机的字体渲染，因此不需要担心字体丢失问题。

## 一些常见误区

- **不要**只改文字不改章节名一致性 —— 改 agenda 必须同步改分章扉页
- **不要**在装饰图上加文字以为是真图表
- **不要**把 lorem ipsum 留在最终 PPT 里
- **不要**为了塞下文字而忽略 `max_chars`
- **不要**修改用户原始模板文件 —— 所有产出都到新文件
- **不要**用本 Skill 做商业项目 —— 见顶部声明
