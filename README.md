# en2zh-plain · 课件大白话

把英文的讲义、课件、教材章节改写成**中文大白话学习笔记**。

术语用「中文译名(English)」双语护航，难点配带失效边界的类比，公式逐符号拆解，每节末尾附考理解的自测题。

这个仓库有两个东西：

| | 是什么 | 给谁用 |
|---|---|---|
| [`skill/`](skill/) | 一个 Claude Code skill | 装在自己电脑上，在终端里用 |
| [`web/`](web/) | 一个网页应用 | 上传文件就能用，不用装任何东西 |

两者共用同一套改写规则。skill 是规则的原本，网页把规则压缩成提示词嵌在页面里。

---

## 为什么要做这个

先搜过一圈，现有方案都只做一半：

- **翻译类 skill**（[claude-translation-skill](https://github.com/senshinji/claude-translation-skill)、[translate-book](https://github.com/deusyu/translate-book)）刻意追求忠实直译，明确不做通俗化改写。
- **通俗化类 skill**（ELI5、[claude-skill-explain](https://github.com/wuchengwei1996/claude-skill-explain)）不做翻译，而且针对单个概念答疑，不处理整份材料。
- 中文社区两个大合集（[claude-code-skills-zh](https://github.com/laolaoshiren/claude-code-skills-zh) 441 个 skill、[education-skills](https://github.com/flysheep-ai/education-skills)）里最接近的只有一个「编程报错英译中」。

核心矛盾是：翻译追求忠实，通俗化要重构表达，两者方向相反。「英文课件 → 中文大白话」需要同时做到，还得保住专业术语的可追溯性。

---

## 核心设计

把英文资料变成中文，难的不是翻译，是**换脑子**。

学术英语靠长句、名词化、被动语态压缩信息。逐词译过来会得到技术上正确、读起来却比原文更累的中文，也就是「翻译腔」。学生读不懂英文原文，读完这种译文照样不懂，还多花了时间。

所以这个 skill 强制的是 **understand → re-express** 两步，不是 translate 一步。中间那步「真正想明白」不能跳。

第二条底线是**大白话不等于说得含糊**。最常见的失败不是太难懂，而是听着顺了、信息却掉光了——把「在 X 条件下成立」简化成「成立」，把 `likely` 说成「就会」，把 `mitigates` 说成「解决」。读者拿着这种笔记去考试会挂科。

完整的规则和背后的理由见 [docs/DESIGN-NOTES.md](docs/DESIGN-NOTES.md)。

---

## 用法一：装成 Claude Code skill

```bash
git clone https://github.com/<your-name>/en2zh-plain.git
cp -r en2zh-plain/skill ~/.claude/skills/en2zh-plain
```

Windows PowerShell：

```powershell
git clone https://github.com/<your-name>/en2zh-plain.git
Copy-Item -Recurse en2zh-plain\skill "$env:USERPROFILE\.claude\skills\en2zh-plain"
```

装好以后，在 Claude Code 里直接丢材料给它：

```
帮我看看这个 lecture09.pdf
把这份英文讲义讲明白
/en2zh-plain
```

材料是 PDF / PPTX / 网页链接 / 粘贴文本都行。输出是一个 HTML 页面。

---

## 用法二：网页版

网页版让不装 Claude Code 的人也能用，拖个文件进去就出笔记。

**自己发布一份：**

1. 在 Claude Code 里打开这个仓库
2. 说：`把 web/index.html 发布成 artifact，capabilities 要 sample 和 downloads`
3. 拿到链接后，从页面右上角 Share 菜单分享给朋友

**为什么必须自己发布：**Artifact 默认私有，别人打不开你的链接。而且页面调用 Claude 用的是**访问者自己的账号额度**，第一次打开会弹窗征求同意——朋友用不会花你的钱，但他们需要有 Claude 账号。

**本地预览：**直接双击 [`web/standalone.html`](web/standalone.html)。文件解析和排版都正常，但**改写功能用不了**——`window.claude` 只在 claude.ai 的 Artifact 环境里存在。页面会在页脚说明这一点。

### 网页版做了什么

| 环节 | 实现 |
|---|---|
| 读 PDF | pdf.js，按文字坐标重建行结构，不是简单拼接 |
| 读 PPTX | JSZip 解压，解析 slide XML，**连演讲者备注一起读** |
| 读 DOCX | 同上，解析 `word/document.xml` |
| 切片 | 优先按原文真实结构切（幻灯片编号、Markdown 标题、章节号），没有结构才按段落堆到 1800 字符 |
| 改写 | 三趟调用：通读定主线和术语表 → 逐节改写（两路并发）→ 收尾 |
| 公式 | 不加载任何数学库，让模型直接输出 Unicode 符号。CSP 问题从根上消失 |
| 安全 | 模型返回的 HTML 过白名单消毒后才进 DOM |

演讲者备注这一条是评测里发现的高价值信息。幻灯片上通常只有裸 bullet，老师「这个每年必考」之类的提示只写在备注里。

---

## 效果

三份不同类型的材料，每份跑两次：带 skill 一次，不带 skill 一次。由独立的评分 agent 对照原文盲打分，每份 15 条评分项。

| 用例 | 带 skill | 对照组 |
|---|---|---|
| 机器学习讲义（公式密集） | 15/15 | 11/15 |
| 经济学教材（密集散文） | 15/15 | 12/15 |
| 生物课件（碎片化 PPT） | 15/15 | 11/15 |
| **合计** | **45/45 (100%)** | **34/45 (75.6%)** |

代价：平均耗时 649 秒对 282 秒，token 98k 对 63k。约 2.3 倍时间、1.6 倍 token。

**但差距在哪里比数字本身重要。**三份材料共同的区分项只有三条：术语速查表（对照组一次都没做）、类比失效边界（对照组从不标注）、术语双语的彻底性。而数字、公式、逻辑严谨度这些**信息保真项，对照组几乎全过**。

结论：不带 skill 也能译对，但**不会每次都把学习脚手架搭齐**。这个 skill 买的是稳定性，不是准确率。

完整方法论、每条评分项的判定依据、以及下一轮该换掉哪些评分项，见 [docs/EVALUATION.md](docs/EVALUATION.md)。

---

## 仓库结构

```
en2zh-plain/
├── skill/                        装进 ~/.claude/skills/ 的部分
│   ├── SKILL.md                  规则原本
│   └── evals/
│       ├── evals.json            3 个用例 × 15 条评分项
│       └── files/                测试材料（ML / 经济学 / 生物）
│
├── web/
│   ├── index.html                Artifact 发布用（body 片段，无 html/head 包裹）
│   └── standalone.html           本地双击可开（由 index.html 生成，别直接改）
│
├── eval/                         评测工具链
│   ├── build-benchmark.ps1       汇总各次评分成 benchmark.json
│   ├── build-review.ps1          生成左右对照的查看器
│   ├── build-standalone.ps1      从 index.html 生成 standalone.html
│   ├── review-template.html      查看器模板
│   ├── review-meta.json          查看器的中文文案与分析要点
│   ├── trigger-evals.json        20 条触发词测试（9 正 11 负）
│   └── results/iteration-1/      第一轮的全部产出与评分
│
└── docs/
    ├── BUILD-LOG.md              这个项目是怎么一步步做出来的
    ├── DESIGN-NOTES.md           每条规则为什么这么写
    ├── EVALUATION.md             评测方法、结果、下一轮改什么
    └── ITERATE.md                怎么跑下一轮迭代
```

---

## 已知限制

- **扫描版 PDF 读不了。**没有文字层的 PDF 会明确报错让你去 OCR，而不是默默输出一份空笔记。
- **老格式不支持。**`.doc` / `.ppt` 需要先另存为 `.docx` / `.pptx`。
- **长材料慢。**单次调用输入上限 64 KiB，并发只有两路。几十页的讲义要跑好几分钟。
- **网页版依赖 claude.ai。**本地打开只能解析文件，不能改写。
- **触发词优化没跑。**需要 Python 环境，开发机上没有。测试集已备好在 `eval/trigger-evals.json`，见 [docs/ITERATE.md](docs/ITERATE.md)。
- **只有一轮评测，每种配置各跑一次。**没有重复采样，所以看不出方差。想要更硬的结论需要每格跑 3 次。

---

## 迭代的时候注意

`SKILL.md` 里有四条规则是**评测中确认有效的**，改之前先看 [docs/DESIGN-NOTES.md](docs/DESIGN-NOTES.md) 里对应的理由：

1. **「读不懂就如实说，别糊一段通顺的话」**——在生物那份材料上触发三次，其中一次直接揪出原讲义一个符号笔误。
2. **公式三步法的「推极端」**——逼出了幻灯片上一个孤立数值的来源。
3. **类比必须标注失效边界**——挡住了把比喻当定义写。
4. **不可牺牲清单里的「逻辑强度」**——`likely` ≠ 一定、`mitigates` ≠ 解决，这些最容易在通俗化过程中被冲掉。

还有一条是**踩过的坑，不要改回去**：公式渲染必须用 MathJax 的 `tex-svg`，**不能用 KaTeX**。Artifact 的内容安全策略只放行 cdnjs 上的脚本，KaTeX 的样式表和字体会被静默拦截，公式会渲染成一堆错位的 span，而且没有任何报错。三个独立的测试执行者全部撞上了这个坑。

---

## 上传到 GitHub

先把 [LICENSE](LICENSE) 里的 `YOUR NAME HERE` 换成你的名字。

**方法一：网页拖拽（不用装任何东西）**

1. 在 GitHub 上新建一个空仓库，**不要**勾选 "Add a README file"
2. 在新仓库页面点 `uploading an existing file`
3. 把 `en2zh-plain` 文件夹**里面的内容**全选拖进去（不是拖文件夹本身）
4. 填提交信息，点 Commit changes

拖拽会保留子目录结构。缺点是每次更新都要重新拖。

**方法二：命令行（需要先装 git）**

这台机器上还没有 git。从 [git-scm.com](https://git-scm.com/download/win) 装完，重开终端：

```powershell
cd "$env:USERPROFILE\Desktop\en2zh-plain"
git init
git add .
git commit -m "en2zh-plain: skill + web app + iteration-1 evals"
git branch -M main
git remote add origin https://github.com/<你的用户名>/en2zh-plain.git
git push -u origin main
```

**提交之前看一眼 `git status`**，确认没有把不该公开的东西带进去。当前仓库里的测试材料都是手写的仿真件，不含真实课件或个人信息。如果你后面加了自己的真实讲义当测试用例，注意有没有版权问题。

---

## License

MIT，见 [LICENSE](LICENSE)。
