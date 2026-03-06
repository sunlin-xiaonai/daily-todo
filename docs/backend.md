# daily-todo 后端设计（Git 文件存储）

目标：

- 不用传统数据库（MySQL / Mongo 等）；
- 使用 **GitHub 仓库中的文件** 作为数据存储；
- 提供一个极简的 HTTP API，供前端读写当天的任务数据；
- 所有人访问同一个链接时，都读写同一份数据。

## 总体架构

- **前端**：`index.html`（GitHub Pages 部署），通过 `fetch` 调用后端 API。
- **后端**：部署在 Vercel 的 Serverless Function（Node.js）。
- **存储**：GitHub 仓库中的 JSON 文件（例如本仓库的 `data/YYYY-MM-DD.json`）。

> 说明：
> - 传统意义上的“数据库”没有使用；
> - 但需要一个极简后端（Serverless Function）来持有 GitHub Token，并通过 GitHub API 读写文件。

## 数据文件格式

路径示例：

```text
/data/2026-03-06.json
/data/2026-03-07.json
...
```

内容结构：

```json
{
  "date": "2026-03-06",
  "tasks": [
    { "id": 1709712345000, "text": "写日报", "completed": true, "createdAt": "2026-03-06T10:00:00.000Z" },
    { "id": 1709712445000, "text": "读书 30 分钟", "completed": false, "createdAt": "2026-03-06T10:02:00.000Z" }
  ]
}
```

## API 设计

统一前缀：`/api/todos`

### GET /api/todos?date=YYYY-MM-DD

- **功能**：获取指定日期的任务列表；如果文件不存在，返回空数组。
- **请求参数**：
  - `date`（可选）：`YYYY-MM-DD`，默认是今天（按服务器时区）。
- **返回示例**：

```json
{
  "date": "2026-03-06",
  "tasks": [
    { "id": 1709712345000, "text": "写日报", "completed": true, "createdAt": "2026-03-06T10:00:00.000Z" }
  ]
}
```

### POST /api/todos

- **功能**：保存指定日期的任务列表到 Git 仓库文件中。
- **请求体** (JSON)：

```json
{
  "date": "2026-03-06",
  "tasks": [
    { "id": 1709712345000, "text": "写日报", "completed": true, "createdAt": "2026-03-06T10:00:00.000Z" }
  ]
}
```

- **行为**：
  - 将上述 JSON 直接写入 `data/2026-03-06.json`；
  - 如果文件不存在则创建，存在则覆盖；
  - 通过 GitHub API 创建一个 commit，commit message 类似：`chore: update todos for 2026-03-06`。

## Vercel 后端示例（Node.js）

在一个 Vercel 项目中创建文件：`api/todos.js`

> 注意：
> - 这里假设你将数据也存放在 **当前仓库**（`sunlin-xiaonai/daily-todo`）的 `data/` 目录中；
> - 也可以改为单独的数据仓库，只需调整 `REPO` 变量。

```js
// api/todos.js

const OWNER = "sunlin-xiaonai";
const REPO = "daily-todo"; // 当前仓库
const BRANCH = "main";
const DATA_DIR = "data";

async function githubRequest(path, options = {}) {
  const token = process.env.GITHUB_TOKEN;
  if (!token) {
    throw new Error("GITHUB_TOKEN is not set");
  }

  const res = await fetch(`https://api.github.com${path}`, {
    ...options,
    headers: {
      "Authorization": `Bearer ${token}`,
      "Accept": "application/vnd.github+json",
      "X-GitHub-Api-Version": "2022-11-28",
      ...(options.headers || {}),
    },
  });

  if (!res.ok) {
    const text = await res.text();
    throw new Error(`GitHub API error ${res.status}: ${text}`);
  }

  return res.json();
}

async function getFileSha(path) {
  try {
    const data = await githubRequest(`/repos/${OWNER}/${REPO}/contents/${encodeURIComponent(path)}?ref=${BRANCH}`);
    return { sha: data.sha, content: data.content, encoding: data.encoding };
  } catch (e) {
    // 文件不存在时返回 null
    return null;
  }
}

export default async function handler(req, res) {
  if (req.method === "OPTIONS") {
    res.setHeader("Access-Control-Allow-Origin", "*");
    res.setHeader("Access-Control-Allow-Methods", "GET,POST,OPTIONS");
    res.setHeader("Access-Control-Allow-Headers", "Content-Type");
    res.status(204).end();
    return;
  }

  res.setHeader("Access-Control-Allow-Origin", "*");

  if (req.method === "GET") {
    const url = new URL(req.url, "http://localhost");
    const date = url.searchParams.get("date") || new Date().toISOString().slice(0, 10);
    const path = `${DATA_DIR}/${date}.json`;

    const file = await getFileSha(path);
    if (!file) {
      res.status(200).json({ date, tasks: [] });
      return;
    }

    const contentJson = Buffer.from(file.content, file.encoding || "base64").toString("utf8");
    res.status(200).json(JSON.parse(contentJson));
    return;
  }

  if (req.method === "POST") {
    const body = req.body;
    if (!body || !body.date || !Array.isArray(body.tasks)) {
      res.status(400).json({ error: "Invalid body" });
      return;
    }

    const date = body.date;
    const path = `${DATA_DIR}/${date}.json`;
    const content = JSON.stringify({ date, tasks: body.tasks }, null, 2);
    const contentBase64 = Buffer.from(content, "utf8").toString("base64");

    const existing = await getFileSha(path);

    const commitMessage = `chore: update todos for ${date}`;

    await githubRequest(`/repos/${OWNER}/${REPO}/contents/${encodeURIComponent(path)}`, {
      method: "PUT",
      body: JSON.stringify({
        message: commitMessage,
        content: contentBase64,
        branch: BRANCH,
        sha: existing ? existing.sha : undefined,
      }),
    });

    res.status(200).json({ ok: true });
    return;
  }

  res.status(405).json({ error: "Method not allowed" });
}
```

### 在 Vercel 上配置

1. 新建一个 Vercel 项目，代码可以来自当前仓库或单独一个后端仓库；
2. 在 Vercel 项目的 **Environment Variables** 中添加：
   - `GITHUB_TOKEN`：一个有权限写 `sunlin-xiaonai/daily-todo` 仓库的 Personal Access Token（PAT）；
3. 部署后，会得到一个类似：
   - `https://your-vercel-app.vercel.app/api/todos` 的地址。

记下这个地址，前端会用它来读写数据。

## 前端集成计划（daily-todo）

接下来对 `index.html` 做的事情：

1. **初始化加载时**：
   - 先从远端 API `GET /api/todos?date=YYYY-MM-DD` 拉取当天任务；
   - 如果拉取失败，则退回到本地 `localStorage`。

2. **在添加 / 修改 / 删除任务后**：
   - 仍然写入 `localStorage`；
   - 同时调用 `POST /api/todos`，把当前完整任务列表同步到远端。

这样：

- 任何访问「同一链接」的人，都会从同一个 JSON 文件读数据；
- 写入时通过后端 + GitHub API 更新该 JSON；
- 数据实际是以文件形式保存在 Git 仓库中，没有使用传统数据库。