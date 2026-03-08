# bib-arxiv-daily

[English README](./README.md)

`bib-arxiv-daily` 会根据你放在本仓库中的 `.bib` 文件，去匹配每天新发布的 arXiv 论文，然后通过 GitHub Actions 定时发送推荐邮件。仓库里还提供了一个“最近 7 天 + 最接近 10 篇”的手动工作流，而这套手动模式的参数本身也是可以扩展的。比如，你可以在最近 `99` 天、最多 `1000` 篇候选论文里，筛出最相关的 `50` 篇：

```bash
.venv/bin/python src/main.py --config config.yaml --lookback-days 99 --max-candidates 1000 --max-results 50 --dry-run --output-html output/manual_99day_top50_report.html
```

所以，这个项目既可以作为一个每天自动发送相关论文邮件的工具，也可以作为一个基于你自己的 `.bib` 馆藏做语义检索和排序的搜索工具。对于“按研究兴趣找论文”这类场景，它通常会比纯关键词加字段过滤更顺手。(arxiv为关键词加字段匹配, 搜索引擎为关键词加语义检索加排名算法)

你既可以在本地手动运行、生成 HTML 报告并直接在浏览器里查看，也可以交给 GitHub Actions 每天定时执行并发送邮件。(也可以本地每日定时运行,了解一下cron)

这个仓库是按“小白可上手”来设计的：

- 你把一个或多个 `.bib` 文件放到 `data/`
- 你在 `config.yaml` 里配置几个 arXiv 分类
- GitHub Actions 每天自动运行
- 工作流把推荐结果以 HTML 邮件发到你的邮箱

当前版本不使用 OpenAI、Claude 或任何付费大模型 API。它在 GitHub Actions 运行器上本地调用开源 embedding 模型。

## 这个项目会做什么

1. 读取 `data/**/*.bib` 下的所有 `.bib` 文件
2. 保留同时包含 `title` 和 `abstract` 的条目
3. 优先抓取你配置的 arXiv 分类 RSS；如果 RSS 临时为空，则退回到 `https://export.arxiv.org/api/query` 查询最近 `24` 小时提交的论文
4. 计算你的馆藏论文和候选论文的文本向量
5. 根据相似度排序
6. 发送 HTML 推荐邮件

你也可以手动触发一个周报流程：直接查询最近 `7` 天提交到 arXiv 的论文，筛出最接近你 bib 库的前 `10` 篇并发送邮件。

当前邮件内容包括：

- 论文标题
- 相似度分数
- 作者
- 摘要片段
- arXiv 链接
- PDF 链接
- 与你 bib 库最接近的几篇论文标题

当前版本不发送 PDF 附件，只发送 PDF 链接。

## 仓库结构

```text
.
├── data/                     # 把一个或多个 .bib 文件放这里
├── src/
│   ├── bib_loader.py
│   ├── arxiv_fetcher.py
│   ├── embedder.py
│   ├── embedding_cache.py
│   ├── recommender.py
│   ├── emailer.py
│   └── main.py
├── config.yaml               # 非私密配置
├── requirements.txt
└── .github/workflows/
    ├── daily.yml
    └── manual-weekly-top10.yml
```

## 开始前你需要准备什么

你需要：

- 一个 GitHub 仓库
- 一个可以通过 SMTP 发信的邮箱
- 一个接收推荐结果的邮箱
- 一个或多个带摘要的 `.bib` 文件

重要提醒：

- `data/*.bib` 会作为仓库内容提交到 Git 历史里。如果你的书目库是私密的，不要放在公开仓库里。
- SMTP 密码、应用专用密码、授权码、收件人邮箱，不要放进 `config.yaml`、代码、截图、issue 或日志里。

## 快速开始

### 1. 把 `.bib` 文件放到 `data/` 目录

支持多个 `.bib` 文件，例如：

```text
data/library.bib
data/reading/ml.bib
data/reading/vision.bib
```

最小可用的 BibTeX 示例：

```bibtex
@article{attention2023,
  title = {A Paper Title},
  abstract = {This abstract is required for similarity matching.},
  author = {Alice Example and Bob Example},
  year = {2023}
}
```

如果某个条目没有 `abstract`，当前版本会直接跳过。

### 2. 修改 `config.yaml`

先从和自己研究方向最相关的少量 arXiv 分类开始，不要一上来配太多。

示例：

```yaml
arxiv:
  categories:
    - cs.LG
    - cs.AI
    - cs.CL
  max_candidates: 80

embedding:
  model: BAAI/bge-small-en-v1.5
  batch_size: 32

ranking:
  top_k_neighbors: 5
  max_results: 15

email:
  subject_prefix: "[arXiv Daily]"
  include_pdf_links: true
  send_empty_email: false

runtime:
  data_dir: data
  output_html: output/latest_report.html
  cache_dir: .cache/recommender
```

给新手的建议：

- 一开始先用 `2` 到 `4` 个分类
- `max_candidates` 先控制在 `50` 到 `100`
- embedding 模型先不要改，先跑通默认值

## 邮件怎么开通：必须打开什么

这个项目当前使用的是普通 SMTP 登录发信，所以你的发件邮箱必须支持 SMTP 发信。

通常需要开启的内容：

- SMTP 服务
- 应用专用密码或授权码
- SSL 或 STARTTLS

常见情况如下。

### Gmail 个人账号

对新手最友好的做法：

1. 给 Google 账号开启两步验证
2. 创建 App Password（应用专用密码）
3. 使用下面这组参数：
   - `SMTP_HOST=smtp.gmail.com`
   - `SMTP_PORT=465`
   - `SMTP_USE_SSL=true`
   - `SMTP_USER=你的 Gmail 地址`
   - `SMTP_PASSWORD=你的 16 位应用专用密码`

Google 官方文档：

- App Passwords: https://support.google.com/mail/answer/185833
- Gmail 接入第三方邮件客户端: https://support.google.com/mail/answer/75726

注意：

- 个人 Gmail 往往是这个仓库最省心的选项。
- 如果你用的是 Google Workspace（公司/学校账号），管理员策略可能要求 OAuth，或者限制应用专用密码。先问管理员。

### Outlook / Microsoft 365

这个仓库当前用的是 SMTP 用户名/密码式登录。对于新的 Microsoft 365 场景，这并不是最稳妥的路线。

按微软 2026-01-27 的官方更新时间线：

- 到 `2026 年 12 月` 前，SMTP AUTH Basic Authentication 行为保持不变
- 到 `2026 年 12 月底`，现有租户会默认禁用它
- 最终彻底移除日期将在 `2027 年下半年` 再公布

微软官方参考：

- 时间线更新：
  https://techcommunity.microsoft.com/blog/exchange/updated-exchange-online-smtp-auth-basic-authentication-deprecation-timeline/4489835
- Microsoft 365 SMTP 文档：
  https://learn.microsoft.com/en-us/Exchange/mail-flow-best-practices/how-to-set-up-a-multifunction-device-or-application-to-send-email-using-microsoft-365-or-office-365

实际建议：

- 如果你是小白，优先选 Gmail 个人邮箱、QQ 邮箱、163 邮箱，或者任何支持授权码的 SMTP 邮箱。
- 如果你必须用 Microsoft 365，后面很可能需要改成 OAuth 方案。

### QQ 邮箱 / 163 邮箱 / 其他邮箱

具体页面会变，但大体流程通常是：

1. 登录网页版邮箱
2. 开启 SMTP 或 POP3/IMAP/SMTP 服务
3. 生成授权码
4. 用这个授权码填 `SMTP_PASSWORD`

如果邮箱同时给你“登录密码”和“授权码”两个选项，应该优先用授权码，不要直接用网页登录密码。

## GitHub Actions 要开哪些东西

### 1. 在仓库里启用 GitHub Actions

打开：

`Settings` -> `Actions` -> `General`

给新手的推荐设置：

- `Actions permissions`: 选择 `Allow all actions and reusable workflows`
- `Workflow permissions`: 选择 `Read repository contents permission`

为什么这样够用：

- 当前工作流只用到了 GitHub 官方动作：`actions/checkout`、`actions/setup-python`、`actions/cache`
- 它不需要把结果写回仓库

GitHub 官方文档：

- 仓库级 Actions 设置：
  https://docs.github.com/github/administering-a-repository/managing-repository-settings/disabling-or-limiting-github-actions-for-a-repository

### 2. 如果这是 fork 出来的仓库，要在 Actions 页面手动启用工作流

GitHub 官方文档说明：

- fork 仓库默认不会自动运行工作流
- public fork 上的定时工作流默认是关闭的
- public 仓库如果长期没有活动，定时工作流也可能在 60 天后被自动禁用

所以你 fork 完之后要做：

1. 打开 `Actions` 页面
2. 点击启用 workflows
3. 如果以后定时任务不跑了，再到 Actions 页面里重新启用

GitHub 官方参考：

- fork 里的工作流事件：
  https://docs.github.com/en/actions/reference/events-that-trigger-workflows
- 工作流启用/禁用：
  https://docs.github.com/actions/managing-workflow-runs/disabling-and-enabling-a-workflow

## 私密信息应该放哪里

放到：

`Settings` -> `Secrets and variables` -> `Actions` -> `Repository secrets`

请创建这些 secrets：

- `SMTP_HOST`
- `SMTP_PORT`
- `SMTP_USER`
- `SMTP_PASSWORD`
- `EMAIL_TO`

可选 secrets：

- `EMAIL_FROM`
- `SMTP_USE_SSL`

推荐规则：

- 只要是密码、token、应用专用密码、授权码、邮箱地址、你不想公开的主机名，都放进 `Repository secrets`
- `config.yaml` 里只保留不敏感的行为配置

当前工作流文件：

- [`.github/workflows/daily.yml`](./.github/workflows/daily.yml)
- [`.github/workflows/manual-weekly-top10.yml`](./.github/workflows/manual-weekly-top10.yml)

当前配置文件：

- [`config.yaml`](./config.yaml)

## Secrets 示例

Gmail 个人账号示例：

```text
SMTP_HOST=smtp.gmail.com
SMTP_PORT=465
SMTP_USER=yourname@gmail.com
SMTP_PASSWORD=your_16_digit_app_password
EMAIL_TO=yourname@gmail.com
EMAIL_FROM=yourname@gmail.com
SMTP_USE_SSL=true
```

## 第一次手动测试怎么做

把文件和 secrets 都准备好后：

1. 打开 `Actions`
2. 打开 `arxiv-daily` 这个 workflow
3. 点击 `Run workflow`
4. 等它跑完

日志里重点看这些信息：

- bib 库是否成功加载
- arXiv 是否成功抓取
- 第一次运行会出现 `Saved ... library embeddings to cache ...`
- 后续运行会出现 `Loaded ... library embeddings from cache ...`
- 邮件是否成功发送

如果没有推荐结果，而且 `send_empty_email: false`，那工作流可以成功结束但不发邮件，这是正常行为。

如果你想跑一次手动周报，而不是普通的日更流程：

1. 打开 `Actions`
2. 打开 `arxiv-weekly-manual-top10` 这个 workflow
3. 点击 `Run workflow`
4. 等它跑完

这个工作流只支持手动触发。它会直接使用 export API 查询最近 `7` 天的 arXiv 提交，最多打分 `500` 篇候选论文，并发送最接近的 `10` 篇。

## 每天什么时候运行

当前定时配置在：

- [`.github/workflows/daily.yml`](./.github/workflows/daily.yml)

现在的 cron 是：

```yaml
schedule:
  - cron: "30 6 * * *"
```

这表示每天 `06:30 UTC` 运行一次。 也就是北京时间下午14:30.

如果你想改时间，直接修改 cron 后提交即可。

## 手动周报工作流

仓库里还包含：

- [`.github/workflows/manual-weekly-top10.yml`](./.github/workflows/manual-weekly-top10.yml)

这个工作流只有 `workflow_dispatch`，不会自动定时运行。

当前固定行为：

- 查询最近 `7` 天提交的 arXiv 论文
- 直接走 export API，不依赖 RSS 当日公告
- 最多打分 `500` 篇候选论文
- 邮件发送最接近的前 `10` 篇

## 当前使用的模型

当前仓库使用的开源 embedding 模型是：

- `BAAI/bge-small-en-v1.5`

它通过 `sentence-transformers` 加载，在 GitHub Actions 运行器本地执行。

要点：

- 这是 embedding 模型，不是对话大模型
- 只用于文本相似度计算
- 不需要 OpenAI API key
- 没有按 token 计费

模型卡：

- https://huggingface.co/BAAI/bge-small-en-v1.5

## 耗时主要受什么影响

主要耗时因素有：

1. 依赖安装
2. PyTorch 安装
3. 首次下载 embedding 模型
4. 你的 `.bib` 中带摘要条目的数量
5. 每天 arXiv 候选论文数
6. GitHub Actions 缓存是否命中
7. GitHub、Hugging Face、arXiv 的网络情况

当前仓库已经缓存了：

- Hugging Face 模型文件
- `.cache/recommender` 下的 library embeddings

这意味着：

- 只要你的 `.bib` 没变，馆藏向量就会复用
- 每天只需要重新算新抓到的 arXiv 候选论文

在标准 `ubuntu-latest` runner 上，粗略经验值是：

- 第一次冷启动：约 `5` 到 `12` 分钟
- 正常热启动且缓存命中：约 `1` 到 `4` 分钟
- 如果 bib 很大，达到几千篇，可能明显更慢

这些是工程经验估计，不是 GitHub 官方保证。

GitHub 官方标准 public runner 规格：

- `ubuntu-latest` 标准 runner：`4 vCPU`、`16 GB RAM`、`14 GB SSD`
- 来源：
  https://docs.github.com/en/actions/reference/github-hosted-runners-reference

## GitHub Actions 免费分钟数

这部分可能会变，所以我这里写的是截至 `2026-03-07` 通过 GitHub 官方文档核对过的数字。

### Public 仓库

使用标准 GitHub-hosted runners 时，public 仓库的 GitHub Actions 是免费且不限量的。

### Private 仓库

标准 GitHub-hosted runners 的每月包含分钟数如下：

| 套餐 | 每月包含分钟数 |
| --- | ---: |
| GitHub Free | 2,000 |
| GitHub Pro | 3,000 |
| GitHub Free for organizations | 2,000 |
| GitHub Team | 3,000 |
| GitHub Enterprise Cloud | 50,000 |

补充说明：

- larger runners 另行计费
- 如果账户没有有效付款方式，超出包含额度后会被阻止继续使用
- artifacts 和 caches 的存储空间也有套餐上限

GitHub 官方参考：

- 计费总览：
  https://docs.github.com/en/billing/managing-billing-for-your-products/managing-billing-for-github-actions/about-billing-for-github-actions
- 套餐包含额度：
  https://docs.github.com/en/billing/reference/product-usage-included

## 给小白的推荐方案

如果你想用最省心的方式跑起来：

1. 如果书目是私密的，就用 private 仓库
2. 先只放少量 `.bib` 文件到 `data/`
3. 先选 `2` 到 `4` 个 arXiv 分类
4. 用 Gmail 个人邮箱 + App Password
5. 先手动触发一次 workflow
6. 第二次运行时确认日志里出现 cache hit

## 本地运行

如果你想在上 GitHub Actions 之前先本地测试：

```bash
python3 -m venv .venv
.venv/bin/python -m pip install -r requirements.txt
.venv/bin/python src/main.py --config config.yaml --dry-run
```

`--dry-run` 会生成 HTML 报告，但不会真正发邮件。

如果你想在本地复现“手动周报 top 10”流程，可以运行：

```bash
.venv/bin/python src/main.py --config config.yaml --lookback-days 7 --max-candidates 500 --max-results 10 --dry-run --output-html output/manual_weekly_top10_report.html
```

几个有用的 CLI 覆盖参数：

- `--lookback-days N`：改成直接查最近 `N` 天的 arXiv 提交，不走 RSS new announcement
- `--max-candidates N`：覆盖 `config.yaml` 里的 `arxiv.max_candidates`
- `--max-results N`：覆盖 `config.yaml` 里的 `ranking.max_results`
- `--output-html PATH`：把 HTML 报告输出到自定义路径

## 常见问题排查

### RSS 没有条目

这里要区分两种情况：

- 周末或节假日，arXiv 本来就可能没有新的 announcement 批次
- RSS 空白期：announcement 已经出了，但 `rss.arxiv.org` 还没同步完成

所以你有时会看到这种现象：

- arXiv 当天的 announcement 已经能看到了
- 但是程序日志里还是 `RSS new papers = 0`
- 再过几个小时，RSS 才恢复正常返回

这就是所谓的 RSS 空白期。

当前仓库已经做了自动兜底：

- 先查 RSS
- 如果 RSS 返回 `0` 个新 id
- 就自动退回到 `https://export.arxiv.org/api/query`
- 用你配置的分类去查最近 `24` 小时的 `submittedDate`

这个兜底可以覆盖 announcement 已出、RSS 还没刷新的那段空白时间；但如果周末确实没有新论文批次，它也不会凭空产生结果。

### 没收到邮件

请检查：

- workflow 是否成功
- 是否因为 `send_empty_email: false` 且当天没有推荐结果
- 发件邮箱是否开启了 SMTP
- 你填的是应用专用密码/授权码，而不是网页登录密码
- 发件邮箱是否拦截了自动化登录

### Workflow 能看到但不会定时跑

请检查：

- 仓库是否启用了 GitHub Actions
- workflow 是否被手动禁用了
- 是否是 fork 仓库导致 schedule 默认关闭
- 是否因为仓库长时间无活动而被 GitHub 自动禁用 schedule

### 第一次运行很慢

这通常是正常的，因为第一次要做：

- 安装依赖
- 下载 embedding 模型
- 构建馆藏 embedding 缓存

### 推荐结果不太准

通常可以从这几个地方改：

- 缩小 arXiv 分类范围
- 降低 `max_candidates`
- 清理质量较差的 `.bib` 条目
- 去掉没有有效摘要的条目

## 当前限制

- 邮件还是 SMTP 登录，不是 OAuth
- 不发送 PDF 附件
- workflow 还没有上传 HTML artifact
- 推荐只基于标题 + 摘要，不是全文
- 没有 `abstract` 的条目会被跳过

## 参考资料

- GitHub Actions 仓库设置：
  https://docs.github.com/github/administering-a-repository/managing-repository-settings/disabling-or-limiting-github-actions-for-a-repository
- GitHub workflow 启用/禁用：
  https://docs.github.com/actions/managing-workflow-runs/disabling-and-enabling-a-workflow
- GitHub Actions 计费：
  https://docs.github.com/en/billing/managing-billing-for-your-products/managing-billing-for-github-actions/about-billing-for-github-actions
- GitHub 套餐包含额度：
  https://docs.github.com/en/billing/reference/product-usage-included
- GitHub-hosted runner 规格：
  https://docs.github.com/en/actions/reference/github-hosted-runners-reference
- Gmail App Passwords：
  https://support.google.com/mail/answer/185833
- Gmail 第三方客户端接入：
  https://support.google.com/mail/answer/75726
- Microsoft 365 SMTP 文档：
  https://learn.microsoft.com/en-us/Exchange/mail-flow-best-practices/how-to-set-up-a-multifunction-device-or-application-to-send-email-using-microsoft-365-or-office-365
- Microsoft SMTP AUTH 时间线更新：
  https://techcommunity.microsoft.com/blog/exchange/updated-exchange-online-smtp-auth-basic-authentication-deprecation-timeline/4489835
