# linkedin-scraper-node

[![运行状态检查](https://github.com/bright-cn/linkedin-scraper-node/actions/workflows/live.yml/badge.svg)](https://github.com/bright-cn/linkedin-scraper-node/actions/workflows/live.yml)
[![最近验证时间](https://img.shields.io/badge/last%20verified-6%20Oct%202026-brightgreen)](https://github.com/bright-cn/linkedin-scraper-node/actions/workflows/live.yml) <!-- verified: rewritten by the daily run -->

[快速开始](#快速开始) · [命令行使用](#作为命令运行) · [API 接口](#其他-api-接口) · [数据](#数据) · [错误处理](#出错时) · [编程智能体](#编程智能体) · [文档](https://docs.brightdata.com/products/scrapers/linkedin/introduction) · [支持](#支持)

使用 JavaScript 将领英个人资料、公司、职位和帖子获取为 JSON。无需登录领英，也无需浏览器。基于 [Bright Data LinkedIn 爬虫 API](https://www.bright.cn/products/web-scraper/linkedin?utm_source=github) 构建。

本项目使用 [Bright Data JavaScript SDK](https://github.com/bright-cn/sdk-js)。完整 API 文档：[LinkedIn 爬虫 API](https://docs.brightdata.com/products/scrapers/linkedin/introduction)。

本仓库还提供用于获取个人资料的单命令 CLI，以及完全无需编写 JavaScript 的 [Bright Data CLI](#编程智能体)。

领英个人资料数据属于个人数据。请遵守适用于你的法律使用这些数据：[Bright Data 合规信息](https://www.bright.cn/legal-governance)。

## 快速开始

需要 Node 20 或更新版本。这个包使用 ESM，因此请使用 `import`，不要使用 `require`。

```bash
npm install @brightdata/sdk
export BRIGHTDATA_API_TOKEN=YOUR_API_KEY
```

从 [Bright Data 控制面板](https://www.bright.cn/cp/setting/users)获取令牌。此 SDK 不会自行读取 `.env` 文件；可以让 Node 通过 `node --env-file=.env yourscript.mjs` 加载。

也可以不手动设置令牌。先运行一次 `npx -p @brightdata/cli bdata login`：它会打开浏览器。此后，SDK 会自行找到已保存的凭据，供你以及在该终端工作的任何编程智能体使用。智能体无法自行完成浏览器中的登录操作，因此请先亲自登录。

还没有账户？[创建账户](https://www.bright.cn/cp/start)；新账户每月可获得 [5,000 免费积分](https://docs.brightdata.com/general/account/billing-and-pricing/free-tier)。

```javascript
import { bdclient } from "@brightdata/sdk";

const client = new bdclient({ autoCreateZones: false });
const job = await client.scrape.linkedin.collectProfiles(
  ["https://www.linkedin.com/in/satyanadella/"],
  { async: true, includeErrors: true },
);
const result = await job.toResult({ pollTimeout: 600_000 });
if (!result.success) throw new Error(`${result.status}: ${result.error}`);
const [profile] = result.data;
console.log(profile.name, "|", profile.position);
console.log(profile.followers, "followers,", profile.connections, "connections");
await client.close();
```

```text
Satya Nadella | Chairman and CEO at Microsoft
12172205 followers, 500 connections
```

预计需要一至三分钟：API 会运行一个任务，`toResult` 则会等待任务完成。每份个人资料消耗 [1 个积分](https://www.bright.cn/pricing/web-scraper)。

上面的代码片段中，以下设置都不可省略。

`autoCreateZones: false` 会阻止 SDK 在启动时创建区域。这些区域用于网络解锁器和搜索引擎 API，是此爬虫工具不会用到的另外两款 Bright Data 产品。如果账户未添加付款方式，创建区域会失败。

`includeErrors: true` 会让 API 把无效的个人资料作为一行结果返回。不设置它，这一行会被丢弃，你将收不到对应结果。

`async: true` 会让 `collectProfiles` 返回任务，而不是使用一分钟后就会放弃的同步接口。

`pollTimeout` 的单位是毫秒，不是秒。如果直接复制 Python 示例中的数字，可能还没进行第一次状态检查就已超时。

## 作为命令运行

此仓库中的命令可以一次处理多份个人资料，并写入一个 JSON 文件。

```bash
npm install -g github:bright-cn/linkedin-scraper-node
linkedin-scraper satyanadella reidhoffman
```

```text
Fetching 2 LinkedIn profiles: satyanadella, reidhoffman
One job for all of them, usually one to three minutes. One credit per profile.
asking  2 profiles...
got     satyanadella: 34 fields (Satya Nadella)
got     reidhoffman: 38 fields (Reid Hoffman)

Saved 2 of 2 profiles as JSON to linkedin.json
```

命令既接受个人资料 URL，也接受其中的路径标识。因此，`linkedin-scraper https://www.linkedin.com/in/satyanadella/` 的效果相同。

在终端中，`asking` 那一行会被下面的进度显示替换，并原地更新，让你知道任务仍在运行以及已经运行了多久：

```text
⠹ 2 profiles 0:01:12
```

```text
--out PATH   output file, default linkedin.json
```

请求的所有个人资料都放在同一个任务中。API 按记录而不是按任务计费；无论任务包含一个 URL 还是十个 URL，通常都需要一至三分钟。因此，获取十份个人资料和获取一份个人资料需要等待的时间大致相同。

也可以导入它，而不是将它作为命令运行。这样，每份个人资料都会得到 `ok` 和 `error` 状态，而不只是原始数据行。单份个人资料失败不会使 `scrape` 拒绝整个请求；读取 `profile` 前请先检查 `ok`：

```javascript
import { scrape } from "@brightdata/linkedin-scraper-node";

for (const outcome of await scrape(["satyanadella", "zz-not-a-real-profile-zz"])) {
  if (outcome.ok) {
    console.log(`${outcome.slug}: ${outcome.profile.name}`);
  } else {
    console.log(`${outcome.slug} failed: ${outcome.error}`);
  }
}
```

```text
satyanadella: Satya Nadella
zz-not-a-real-profile-zz failed: The profile is hidden or private.
```

## 其他 API 接口

上面的命令对应下表第一行。其余各行是 SDK 提供的其他领英接口，详见 [LinkedIn 爬虫 API 文档](https://docs.brightdata.com/products/scrapers/linkedin/introduction)。下面的每段代码都是完整示例，只需要 `@brightdata/sdk`，可直接粘贴运行。所有示例每周一都会在 Actions 中运行，其他日子还会执行规模较小的检查。页面顶部的徽章显示最近一次结果。

| 已有信息 | 想获取 | 调用方式 |
| --- | --- | --- |
| 个人资料 URL | 对应个人资料 | `collectProfiles([url], { async: true, includeErrors: true })` |
| 名和姓 | 匹配的个人资料 | `discoverProfiles([{ first_name, last_name }], …)`；见下文说明 |
| 公司 URL | 对应公司 | `collectCompanies([url], { async: true, includeErrors: true })` |
| 职位 URL | 对应职位信息 | `collectJobs([url], { async: true, includeErrors: true })` |
| 关键词和地点 | 匹配的职位信息 | `discoverJobs([{ location, keyword }], …)` |
| 帖子 URL | 对应帖子 | `collectPosts([url], { async: true, includeErrors: true })` |
| 个人资料 URL | 此人发布的帖子 | `discoverUserPosts([{ url }], …)` |
| 公司 URL | 该公司发布的帖子 | `discoverCompanyPosts([{ url }], …)` |

这些方法都位于 `client.scrape.linkedin` 下。

`discoverProfiles` 接受 `{ first_name, last_name }`，而不是 URL。它按姓名搜索人员。本 README 只说明此功能的存在，不会以真实人物演示。

这里也没有使用固定职位 URL 的示例。职位可能停止招聘；如果示例固定引用该职位，它关闭的那一周，检查结果就会变红。请按表中所示，改用关键词和地点发现职位。

上述调用都会创建异步任务：API 触发任务，SDK 轮询状态，并在任务就绪后返回。这就是一次调用需要一至三分钟的原因。API 的[同步接口](https://docs.brightdata.com/api-reference/scrapers/synchronous-requests)仅供直接通过 HTTP 调用，最多接受 20 个 URL，且有一分钟的时间限制。

`profiles`、`companies`、`jobs` 和 `posts` 这四个简短方法名会在一次调用中完成触发、轮询和获取，并返回 `ScrapeResult`。名称以 `collect` 或 `discover` 开头的方法会返回需要自行轮询的 `ScrapeJob`。只有后一组方法可以传入 `includeErrors`。如果需要区分无效个人资料与空结果，请使用后一组方法并传入该参数。

### 多份个人资料，一个任务

一个 URL 数组只会创建一个任务，不会为每份个人资料各建一个任务。

```javascript
import { bdclient } from "@brightdata/sdk";

const client = new bdclient({ autoCreateZones: false });
const job = await client.scrape.linkedin.collectProfiles(
  [
    "https://www.linkedin.com/in/satyanadella/",
    "https://www.linkedin.com/in/reidhoffman/",
  ],
  { async: true, includeErrors: true },
);
const result = await job.toResult({ pollTimeout: 600_000 });
if (!result.success) throw new Error(`${result.status}: ${result.error}`);
for (const profile of result.data) {
  console.log(profile.name, "|", profile.city ?? "no city", "|", profile.followers, "followers");
}
await client.close();
```

```text
Reid Hoffman | United States | 2792708 followers
Satya Nadella | Redmond, Washington, United States | 12172206 followers
```

返回结果的顺序不作保证。请用 `input_url` 匹配请求的个人资料，不要依赖行的位置。

### 现在触发，稍后获取

如果要处理的不止几份个人资料，不要让进程阻塞一小时。先触发任务并保存快照 ID，待任务就绪后再获取结果。快照可在 30 天内下载。

```javascript
import { bdclient } from "@brightdata/sdk";

const client = new bdclient({ autoCreateZones: false });
const job = await client.scrape.linkedin.collectProfiles(
  ["https://www.linkedin.com/in/satyanadella/"],
  { async: true, includeErrors: true },
);
console.log("snapshot:", job.snapshotId);
await job.wait({ pollInterval: 5_000, pollTimeout: 600_000 });
console.log("status:", await job.status());
const [record] = await job.fetch();
console.log("fetched:", record.name, "|", record.current_company_name);
await client.close();
```

```text
snapshot: sd_mu52fz8odk79uf5we
status: ready
fetched: Satya Nadella | Microsoft
```

`ScrapeJob` 还提供 `download()` 方法，可将快照写入磁盘；也提供 `cancel()` 方法，用于停止任务。

### 获取公司数据

```javascript
import { bdclient } from "@brightdata/sdk";

const client = new bdclient({ autoCreateZones: false });
const job = await client.scrape.linkedin.collectCompanies(
  ["https://www.linkedin.com/company/bright-data/"],
  { async: true, includeErrors: true },
);
const result = await job.toResult({ pollTimeout: 600_000 });
if (!result.success) throw new Error(`${result.status}: ${result.error}`);
const [company] = result.data;
console.log(company.name, "|", company.employees_in_linkedin, "employees on LinkedIn");
await client.close();
```

```text
Bright Data | 407 employees on LinkedIn
```

### 按关键词和地点发现职位

`location` 是必填项。`keyword`、`company`、`time_range`、`job_type`、`experience_level` 和 `remote` 是可选项。

```javascript
import { bdclient } from "@brightdata/sdk";

const client = new bdclient({ autoCreateZones: false });
const job = await client.scrape.linkedin.discoverJobs(
  [{ location: "London", keyword: "data engineer", time_range: "Past week" }],
  { async: true, includeErrors: true, limitPerInput: 5 },
);
const result = await job.toResult({ pollTimeout: 900_000 });
if (!result.success) throw new Error(`${result.status}: ${result.error}`);
for (const posting of result.data.slice(0, 3)) {
  console.log(posting.job_title, "|", posting.company_name, "|", posting.job_location);
}
await client.close();
```

```text
Data Engineer | FDJ UNITED | London Area, United Kingdom
Senior Data Engineer | Firstup | London, England, United Kingdom
Data Engineer | General Atlantic | London Area, United Kingdom
```

职位发现功能按行计费，每行消耗 1 个积分；宽泛的筛选条件可能匹配数千条职位信息。`limitPerInput` 可限制返回数量。上面的调用实际返回了 5 条。

请同时传入 `async: true`。如果缺少它，`limitPerInput` 会在请求发送前被移除，且不会报错，导致任务不受数量上限约束（[sdk-js#36](https://github.com/bright-cn/sdk-js/issues/36)）。SDK 的 `DiscoverOptions` 类型却提示省略 `async`，因此符合其类型定义的调用反而可能产生更多费用。

## 数据

大多数人关心的个人资料字段：

```text
name  position  city  current_company_name  followers  connections  experience
```

代码没有硬编码字段列表。API 返回的所有字段都会进入 `result.data`，也会写入命令生成的文件。

API 在数据结构中以 `pii: true` 将其中 8 个字段标记为个人数据：`id`、`name`、`about`、`url`、`input_url`、`linkedin_id`、`first_name` 和 `last_name`。请读取该标记，而不是自行维护字段列表。

<!-- fields:start -->
<details>
<summary>全部 47 个字段及其类型和说明</summary>

以下内容每天都会通过 `client.datasets.linkedinProfiles.getMetadata()` 从数据集的数据结构重新生成，因此不会过时。每份个人资料只包含适用于它的字段：示例文件包含这 47 个字段中的 32 个，另有数据结构未列出的 `timestamp` 和 `input`。

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | 文本 | 个人数据。此人领英个人资料的唯一标识符 |
| `name` | 文本 | 个人数据。个人资料名称 |
| `city` | 文本 | 用户的地理位置 |
| `country_code` | 文本 | 用户地理位置对应的国家或地区代码 |
| `position` | 文本 | 个人资料中显示的当前职位或职称 |
| `about` | 文本 | 个人数据。简短的个人资料简介。网站有时只显示带有“…”的截断版本，此时采集到的也是该版本 |
| `posts` | 数组 | 用户近期领英帖子的相关信息，通常包括帖子标题、创建日期、帖子 URL 等 |
| `groups` | 数组 | 此人加入的领英群组 |
| `current_company` | 对象 | 用户当前工作情况的信息，通常包括公司名称、职位、公司 ID 和所属行业 |
| `experience` | 数组 | 用户的职业经历，通常包括职位、任职时长、公司所在地、起止日期、公司名称和公司资料页 URL 等 |
| `url` | URL | 个人数据。直达领英个人资料的 URL |
| `people_also_viewed` | 数组 | 浏览过此用户个人资料的人还浏览过的领英个人资料列表 |
| `educations_details` | 文本 | 用户的教育背景信息 |
| `education` | 数组 | 用户的教育背景，通常包括学位、起止年份、专业领域等 |
| `recommendations_count` | 数字 | 用户收到的推荐总数 |
| `avatar` | URL | 领英用户头像的 URL |
| `courses` | 数组 | 用户修读的课程或教育项目 |
| `languages` | 数组 | 用户掌握不同语言的情况 |
| `certifications` | 数组 | 执照和认证 |
| `recommendations` | 数组 | 用户在领英上收到的联系人或同事推荐 |
| `volunteer_experience` | 数组 | 用户的志愿服务经历 |
| `followers` | 数字 | 关注该个人资料的用户或公司数量 |
| `connections` | 数字 | 此人的领英联系人数量 |
| `current_company_company_id` | 文本 | 此人最近或当前所在公司的 ID |
| `current_company_name` | 文本 | 此人最近或当前所在公司的名称 |
| `publications` | 数组 | 已发表的作品或演讲 |
| `patents` | 数组 | 已申请或已授权的专利 |
| `projects` | 数组 | 职业或学术项目 |
| `organizations` | 数组 | 加入的专业组织 |
| `location` | 文本 | 用户的地理位置 |
| `input_url` | URL | 个人数据。开始抓取时输入的 URL |
| `linkedin_id` | 文本 | 个人数据。领英个人资料标识符 |
| `activity` | 数组 | 用户与帖子相关的活动 |
| `linkedin_num_id` | 文本 | 数字形式的领英个人资料 ID |
| `banner_image` | URL | 横幅图片 |
| `honors_and_awards` | 数组 | 获得的荣誉与奖项 |
| `similar_profiles` | 数组 | 与当前个人资料类似的资料 |
| `default_avatar` | 布尔值 | 头像是否为默认的空白头像 |
| `memorialized_account` | 布尔值 | 账户是否已设为纪念账户 |
| `bio_links` | 数组 | 添加到个人简介中的外部链接 |
| `first_name` | 文本 | 个人数据。用户的名 |
| `last_name` | 文本 | 个人数据。用户的姓 |
| `urn_id` | 文本 | 领英使用的统一资源名称（URN）标识符 |
| `urn` | 文本 | 统一资源名称 |
| `influencer` | 布尔值 | 个人资料是否标记为有影响力人物 |
| `fsd_profile_id` | 文本 | FSD 个人资料 ID |
| `backfilled_columns` | 对象 | 标识固定字段是否经过回填。键为字段名，值为 `true` 或 `false` |

</details>
<!-- fields:end -->

<details>
<summary>真实输出文件的开头，来自 <code>linkedin-scraper satyanadella</code></summary>

```json
{
  "generated_at": "2026-09-17T05:07:02.996Z",
  "profiles": [
    {
      "slug": "satyanadella",
      "profile": {
        "id": "satyanadella",
        "name": "Satya Nadella",
        "city": "Redmond, Washington, United States",
        "country_code": "US",
        "position": "Chairman and CEO at Microsoft",
        "about": "As chairman and CEO of Microsoft, I define my mission and that of my company as empowering every person and every organization on the planet to achieve more.",
        "posts": [
          {
            "title": "How do we build a frontier intelligence ecosystem?",
            "attribution": "Great to be back at Microsoft Build today. For us, it is not about any one piece of technology or even the platform.",
            "img": "https://media.licdn.com/dms/image/v2/D5612AQEiPOCznzusVw/article-cover_image-shrink_720_1280/B56ZnaEmAPHAAI-/0/1760300263776?e=2147483647&v=beta&t=wFogtGsELbNXfbXZIw37bNUzyYGF8CNcMpj-BMIKqpI",
            "link": "https://www.linkedin.com/pulse/how-do-we-build-frontier-intelligence-ecosystem-satya-nadella-73jhc",
            "created_at": "2026-06-02T00:00:00.000Z",
  ...
```

包含一份个人资料及其全部字段的完整文件见 [examples/sample_output.json](examples/sample_output.json)。

</details>

## 出错时

| 看到的信息 | 含义 |
| --- | --- |
| `API token required but not found.` | 在发送任何请求之前以状态码 2 退出。请设置令牌。 |
| `failed  slug: ...` | 以状态码 1 退出。个人资料不存在，通常是路径标识输入有误。 |
| `failed  slug: the API returned no row for this profile` | 以状态码 1 退出。任务结果中没有该输入对应的数据行。请重新运行。 |
| `failed  slug: Polling timed out after 605s for sd_...` | 以状态码 1 退出。请求在等待 600 秒后放弃。API 运行缓慢时可能出现这种情况；请重新运行。 |

任何失败都会以状态码 1 退出，因此脚本可以安全地依据运行结果决定是否继续。

在 SDK 中，相同情况表现如下：

| 看到的信息 | 含义 |
| --- | --- |
| `AuthenticationError: No API token found.` | 在选项、环境变量和 CLI 登录信息中都找不到令牌。 |
| 状态码为 401 的 `APIError` | 已设置令牌，但令牌不正确。 |
| `result.success` 为 `false`，`result.status` 为 `"timeout"` | SDK 等待超时。增大以毫秒为单位的 `pollTimeout`，或重新运行。 |
| `result.data` 中某一行包含 `error` 键 | 这是启用 `includeErrors` 后 API 对某条输入返回的结果；其他行不受影响。 |
| 原本预计会返回错误，却得到零行 | 没有传入 `includeErrors: true`，因此 API 丢弃了原本会说明错误原因的那一行。 |

## 编程智能体

无需编写 JavaScript，也无需预先安装任何内容。粘贴以下两行即可；第一行会打开一次浏览器。如果通过 SSH 或在 CI 中使用，请改用 `bdata login --device`：

```bash
npx -p @brightdata/cli bdata login
npx -p @brightdata/cli bdata pipelines linkedin_person_profile "https://www.linkedin.com/in/satyanadella/"
```

`bdata pipelines list` 会列出所有类型。领英相关类型包括 `linkedin_person_profile`、`linkedin_company_profile`、`linkedin_job_listings`、`linkedin_posts` 和 `linkedin_people_search`。它们接受 URL、输出 JSON，并按每条记录 1 个积分计费。

运行 `npx skills add brightdata/skills`，可以让 Claude Code、Cursor 和 Codex 学会这些命令并了解相关文档，此后就能用自然语言提出需求。完整指南：[让编程智能体使用 Bright Data](https://docs.brightdata.com/quickstart-coding-agent)。

使用托管助手、没有终端？[Bright Data MCP 服务器](https://github.com/bright-cn/brightdata-mcp#which-tool-to-use)的 `social` 工具组中包含领英工具；该工具组默认关闭，需要明确启用：

```text
https://mcp.brightdata.com/mcp?token=YOUR_API_TOKEN&groups=social
```

智能体还可以自行创建账户，无需填写注册表单：[智能体注册](https://www.bright.cn/auth.md)。如需了解 Bright Data 与 LangChain、Zapier、n8n 等工具的其他集成方式，请参阅[集成文档](https://docs.brightdata.com/integrations/introduction)。

## 支持

发现此仓库中的问题？请[提交 issue](https://github.com/bright-cn/linkedin-scraper-node/issues)，并参照 [CONTRIBUTING.md](CONTRIBUTING.md) 说明提供必要信息。

有关 API、账户或积分的问题，请联系 [Bright Data 支持团队](https://brightdata.zendesk.com/hc/en-us/requests/new)。

## 许可证

MIT。
