# interview_note · Claude 协作指南

> 这份文件给 Claude（以及未来的我）看，定义这个仓库的**写作规范、文件约定、链接格式、维护原则**。
>
> 个人偏好见 `~/.claude/CLAUDE.md`（全局）和 `~/.claude/projects/-Users-nikki-go-src-interview-note/memory/`（项目记忆）。

---

## 一、项目定位

- **目标**：资深 Go 后端工程师（5 年以上，对标 P7 / TL / 技术专家）面试知识体系
- **范围**：技术深度 + 工程实战 + 软实力，不是入门教程
- **使用场景**：
  - 主用：**Obsidian 本地浏览编辑** ← 跳转格式以 Obsidian 为准
  - 备份：GitHub（公开仓库 + 同步）

---

## 二、目录结构

```
01-go-language/         Go 语言（细粒度，按 API 拆分）
02-os/                  操作系统
03-mysql/               MySQL（按原理 + 场景聚合）
04-redis/               Redis（按原理 + 场景聚合）
05-message-queue/       消息队列
06-distributed/         分布式系统
07-microservice/        微服务
08-architecture/        架构
09-ddd/                 领域驱动设计
10-system-design/       系统设计
11-cdn/                 CDN（用户强项）
12-ai/                  AI / LLM
13-engineering/         工程化
14-projects/            项目讲述（面试用）
15-leadership/          软实力（P7+ 必备）
16-software-craftsmanship/ 软件工艺
99-meta/                速记题集 / 跨主题索引
```

**未来扩展位**（按需新增）：

```
17-learning-log/        学习日志 / 面试反馈
18-industry-trends/     业界动态跟踪
19-blog-drafts/         技术品牌输出（博客 / 演讲）
```

---

## 三、文件命名约定

| 文件名模式 | 含义 | 示例 |
| --- | --- | --- |
| `00-XX-map.md` | 目录总览（知识树 / 题型分级 / 学习路径）| `00-redis-map.md` |
| `NN-XX.md` | 主题文件（NN 是 01-99 的二位数）| `02-data-structures.md` |
| `*-senior-answers.md` | 资深答题模板（四段式）| `21-senior-interview-answers.md` |
| `*-go-practice.md` | Go SDK 工程落地 | `20-go-redis-best-practices.md` |
| `*-design-tradeoffs.md` | 设计边界 / 反问视角 | `19-redis-design-tradeoffs.md` |
| `*-internals.md` | 源码深水区 | `22-innodb-internals.md` |
| `*-antipatterns*.md` | 反模式 / 危险操作 | `23-antipatterns-dangerous-ops.md` |
| `99-meta/XX-20.md` | 速记 20 题 | `99-meta/redis-20.md` |

---

## 四、内容标准（资深视角）

### 4.1 必含元素

每个深度文件都应该有：

- **原理 / 机制**：源码级别 / 数据结构 / 算法
- **场景 / 用法**：什么时候用、怎么用、Go 代码示例
- **代价 / 边界**：什么时候不能用、性能损失、复杂度
- **量化数据**：QPS / 延迟 / 容量 / 成本
- **实战案例**：真实生产故障 + STAR 复盘
- **答题模板**：一句话 → 分层 → 边界 → 案例

### 4.2 答题模板（四段式 STAR-L）

```
S 背景    Situation    业务规模 / 团队 / 角色
T 任务    Task         目标 / 痛点 / 限制
A 行动    Action       方案对比 + 选定 + 实施 + 踩坑（70% 篇幅）
R 结果    Result       多维量化（性能 + 成本 + 业务）
L 反思    Learnings    取舍 / 失败 / 成长（区分资深关键）
```

### 4.3 资深信号（写作时主动加入）

- ✓ 主动讲取舍（不是只讲选定方案）
- ✓ 主动讲代价（性能 vs 成本 / 一致 vs 可用）
- ✓ 主动讲边界（什么时候这套不行）
- ✓ 主动讲失败（踩过的坑 + 反思）
- ✓ 主动讲演进（X 版本前 / 后变化）
- ✓ 量化数据（不模糊"挺大的"）

### 4.4 视觉规范

- **Mermaid** 图用于流程 / 时序 / 架构 / mindmap
- **表格**用于对比 / 选型
- **代码块**用于 Go / SQL / shell / 配置（必须带语言标签 `\`\`\`go`）
- **纯文本块**（无语言标签）尽量少用，会影响 Obsidian 渲染

---

## 五、链接规范（关键）

### 5.1 不同场景用不同格式

| 场景 | 格式 | 示例 |
| --- | --- | --- |
| **同目录跳转** | `[name.md](name.md)` | `[02-index.md](02-index.md)` |
| **跨目录跳转** | `[name.md](../dir/name.md)` | `[../03-mysql/02-index.md](../03-mysql/02-index.md)` |
| **目录跳转** | `[dir/](../dir/)` | `[../09-ddd/](../09-ddd/)` |
| **99-meta/*-20.md 内的 TOC** | **`[[#标题]]`** wiki link | `[[#1. CDN 解决什么问题？三层架构？]]` |

### 5.2 Obsidian vs GitHub 取舍

**事实**（已查证）：
- Obsidian **不支持** `[text](#slug)` 标准 markdown 锚点（被归 Bug graveyard）
- Obsidian **不支持** `<a id="qN">` HTML 锚点
- GitHub **不渲染** `[[#标题]]` wiki link

**因此**：
- 99-meta/*-20.md 这种"目录 → 题目"跳转用 **wiki link**（Obsidian 优先）
- 其他文件之间的跳转用 **markdown 标准链接**（双兼容，GitHub 渲染正常）

### 5.3 反模式（禁止）

```
❌ 在代码块里嵌 markdown 链接 → GitHub 不渲染
❌ "详见 04-redis/05" 裸数字简写 → 不可点击
❌ "对应：05-cache-patterns" 没加 .md 和方括号
❌ HTML id `<a id="qN">` 加在标题里 → Obsidian 不识别
```

---

## 六、Git 规范

### 6.1 提交粒度

- **单知识点单 commit**（即使一个 commit 改 200 行）
- commit message 格式：`<dir>: <file-or-topic> - <change>`
- 例：
  ```
  redis: 20-go-redis-best-practices - go-redis 工程落地
  mysql: 16-go-mysql-practice 大补 252→1000+ 行
  fix(redis): 重画跳表图（同节点垂直贯穿多层）
  ```

### 6.2 安全规则

- **NEVER** force push main / master
- 优先用 `--force-with-lease` 而不是 `--force`
- commit 前 `git diff --staged` 检查
- `.obsidian/workspace.json` 是本地状态，**不要提交**（用 stash）
- 跨平台兼容：用 `git stash push -m local-obsidian -- .obsidian/workspace.json`

### 6.3 push 流程

```bash
# 标准流程
git add <specific-files>          # 不要 git add -A（防 .obsidian 误提）
git commit -m "..."
git stash push -m local-obsidian -- .obsidian/workspace.json
git pull --rebase origin main
git push origin main
git stash pop                     # 恢复本地 obsidian 状态
```

---

## 七、维护原则

### 7.1 高内聚优先于去重

- 宁可重复，让每个块**自包含**
- 每篇独立可读，不强制跨文件跳转
- 跳转只用于"想深入了解"

### 7.2 知识点粒度按领域差异化

- **Go**：每 API / 概念一篇（`slice.md` / `map.md` / `string.md`）
- **Redis / MySQL**：按原理 + 场景聚合（不按命令切）
- **分布式 / 架构**：按主题（事务 / 锁 / 限流）

### 7.3 持续维护节奏

```
每周：把当周学的东西归档（如有）
每月：检查"待补"项，补 1-2 个
每季度：知识树 review，删过时内容
每半年：检查链接有效性，修内联引用
每年：大盘整理，决定下个阶段重点
```

### 7.4 不做的事

- ❌ 不写"教程"（这是面试知识库，不是入门书）
- ❌ 不堆术语（讲清楚为什么 + 代价）
- ❌ 不抄文档（要有自己的总结 + 实战）
- ❌ 不追求面面俱到（高 ROI 优先）

---

## 八、Claude 协作期望

### 8.1 我希望 Claude 帮我做什么

✅ **批量改造**：跨文件统一格式 / 锚点 / 链接（脚本验证后再执行）
✅ **新文件起草**：按规范生成完整框架，我来填实战细节
✅ **审计**：定期扫描跳转死链 / 命名不一致 / 缺失答题模板
✅ **重构**：把过长文件拆分 / 把零散内容聚合
✅ **校对**：发现错别字、术语错误、过时信息

### 8.2 我不希望 Claude 做什么

❌ **未经测试就批量改代码块**（之前破坏过 mermaid，必须先小样本验证）
❌ **自动加 emoji**（除非我明确要求）
❌ **自动写"教程感"很重的内容**（要"资深视角"）
❌ **沉默工作**（修改后必须报告改了什么 + 量化）

### 8.3 上下文复用

如果 Claude 不确定某个写作风格，参考以下"金标准"文件：

| 类别 | 参考文件 |
| --- | --- |
| 总览地图 | `04-redis/00-redis-map.md` |
| 答题模板 | `04-redis/21-senior-interview-answers.md` |
| Go 实战 | `04-redis/20-go-redis-best-practices.md` |
| 设计边界 | `04-redis/19-redis-design-tradeoffs.md` |
| 源码深水区 | `04-redis/17-object-encoding-internals.md` |
| 反模式 | `03-mysql/23-antipatterns-dangerous-ops.md` |
| 项目故事 | `14-projects/07-ecommerce-story.md` |
| Leadership | `15-leadership/04-senior-answers.md` |

新文件以这些为模板，能减少风格漂移。

---

## 九、长期愿景（路线图）

### 9.1 短期（1-3 月）

- [ ] 建 `17-learning-log/` 学习日志（按月归档）
- [ ] 建 `99-meta/wrong-answers.md` 错题本
- [ ] 每个 14-projects 文件加上"我的真实经历"占位章节
- [ ] 全目录链接死链审计

### 9.2 中期（3-12 月）

- [ ] 建 `14-projects/personal/` 个人真实项目库（脱敏后写）
- [ ] 建 `99-meta/mock-interview/` 模拟面试记录
- [ ] 用 mkdocs / Vitepress 做在线版本
- [ ] 建 `99-meta/cross-topic-flashcards/` Anki 卡片

### 9.3 长期（1+ 年）

- [ ] 公开版（脱敏后开源 / 技术博客）
- [ ] 写一本"Go 后端面试体系"小册
- [ ] 演讲 / 大会分享素材库
- [ ] 知识图谱可视化（Obsidian Graph 优化）

---

## 十、紧急参考

如果跳转 / 格式 / 链接出问题：

| 问题 | 解决 |
| --- | --- |
| Obsidian 报"未找到 #xxx" | TOC 改 `[[#标题原文]]` wiki link |
| GitHub 链接显示成字面文本 | 不要在代码块里嵌 markdown 链接 |
| `mermaid` 图被破坏 | 检查 `\`\`\`mermaid` 后的内容、`\`\`\`` 闭合 |
| 跨目录链接 404 | 检查 `../` 层级，用 `python os.path.relpath` 验证 |
| `.obsidian/workspace.json` 误提交 | `git rm --cached .obsidian/workspace.json` |

---

**Last updated**: 2026-05-10
