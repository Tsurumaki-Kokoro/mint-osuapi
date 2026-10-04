# MintOsuAPI

**简体中文** | [English](README.en.md)

[![.NET 8+](https://img.shields.io/badge/.NET-8%2B-512BD4)](src/MintOsuApi/MintOsuApi.csproj) [![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
MintOsuAPI 是一个面向 .NET 的 osu! API 客户端，为机器人、桌面应用和后端服务提供强类型的数据访问接口。从 [MintAPI](https://github.com/Tsurumaki-Kokoro/MintAPI) 抽离，可独立使用。

## 特性

- **v2 与 v1**：OAuth2 v2 API 和基于 API Key 的旧版 v1 API。
- **强类型模型**：用户、成绩、谱面、排名和多人比赛等数据，以及对应的枚举和响应模型。
- **自动认证**：内置公共数据访问令牌获取、磁盘缓存及 401 后的一次刷新。
- **异步调用**：通过 `async` / `await` 使用，支持 `CancellationToken`。
- **数据兼容**：处理新旧成绩字段、不同 Mods 格式和多态响应。
- **自定义 HTTP**：可传入预先配置的 `HttpClient`。

目标框架为 **.NET 8**，支持 .NET 8 及更高版本。库本身不依赖 MintAPI、数据库、Redis 或浏览器。

## 安装

通过 NuGet 安装：

```sh
dotnet add package MintOsuAPI
```

本地开发也可直接引用源码项目，在消费项目目录执行：

```sh
dotnet add reference /path/to/mint-osuapi/src/MintOsuApi/MintOsuApi.csproj
```

包名为 `MintOsuAPI`，代码命名空间为 `MintOsuApi`。

## 快速开始

在 osu! 账户设置中创建 OAuth 应用，取得客户端 ID 和密钥。以下示例从环境变量读取凭据：

### API v2（OAuth2 客户端凭据）

```csharp
using MintOsuApi;
using MintOsuApi.Enums;
using MintOsuApi.Models;

int clientId = int.Parse(Environment.GetEnvironmentVariable("OSU_CLIENT_ID")!);
string clientSecret = Environment.GetEnvironmentVariable("OSU_CLIENT_SECRET")!;

using var client = new OsuClient(clientId, clientSecret);

User user = await client.GetUserAsync("peppy", GameMode.Osu);
Beatmap map = await client.GetBeatmapAsync(beatmapId: 75);
List<Score> scores = await client.GetUserScoresAsync(
    user.Id, ScoreType.Best, mode: GameMode.Osu, limit: 10);
```

`OsuClient` 自动获取访问令牌并将其缓存到磁盘，缓存令牌即将过期时才会重新获取。默认目录在 Windows 上为 `%APPDATA%\mintosuapi\tokens`，其他平台通过 .NET 的 `ApplicationData` 目录解析（通常为 `~/.config/mintosuapi/tokens`）；可通过 `tokenDirectory` 参数覆盖。

缓存键由客户端 ID、密钥和授权范围等信息计算而成，与目录路径无关，因此移动缓存目录不会改变缓存键。不同凭据的应用应分别使用独立的 `TokenStore` 目录。

### 使用自定义 HttpClient

需要接入依赖注入或自定义 HTTP 配置时，可传入预先配置的 `HttpClient`。以下配置与默认构造函数的认证方式一致：

```csharp
using MintOsuApi;
using MintOsuApi.Auth;

var authHandler = new OsuAuthHandler(
    clientId, clientSecret,
    "https://osu.ppy.sh/oauth/token",
    new TokenStore())
{
    InnerHandler = new HttpClientHandler()
};

var http = new HttpClient(authHandler)
{
    BaseAddress = new Uri("https://osu.ppy.sh/api/v2/")
};
http.DefaultRequestHeaders.Add("User-Agent", "MyApp/1.0");

using var client = new OsuClient(http);
```

该构造函数会补充 `x-api-version` 请求头，`User-Agent` 需要自行设置为能标识应用的名称。释放 `OsuClient` 时也会释放传入的 `HttpClient`，管理其生命周期时需要考虑这一行为。

内置认证处理器使用客户端凭据授权和 `public` 范围。聊天、论坛写入、好友、`GetMeAsync` 等需要用户授权的接口，必须使用具有相应授权的 `HttpClient`；默认构造函数不执行授权码流程。

### API v1（旧版 API Key）

```csharp
using MintOsuApi.V1;

string apiKey = Environment.GetEnvironmentVariable("OSU_API_KEY")!;

using var client = new OsuV1Client(apiKey);

V1User? user = await client.GetUserAsync("peppy");
List<V1Score> best = await client.GetUserBestAsync("peppy", limit: 50);
string replayData = await client.GetReplayAsync(beatmapId: 75, user: "peppy");
```

## 错误处理

| 异常 | 来源 | 含义 |
|---|---|---|
| `OsuApiException` | `OsuClient` | v2 返回 API 错误载荷 |
| `OsuApiV1Exception` | `OsuV1Client` | v1 返回错误，或 API Key 被拒绝 |
| `V1ReplayUnavailableException` | `OsuV1Client.GetReplayAsync` | 成绩没有可用的回放 |

v2 收到非成功 HTTP 状态码时，会先抛出 `HttpRequestException`，不会进入 API 错误载荷解析。

上述 API 专用异常分别位于 `MintOsuApi` 和 `MintOsuApi.V1` 命名空间。

## 兼容性说明

**多人比赛中的旧版成绩。** `MintOsuApi.Models.MatchGame.Scores` 的类型为 `List<LegacyScore>`。专用转换器同时接受旧字段 `score` 和新字段 `total_score`，两者同时出现时优先使用 `total_score`。从早期版本迁移时，应读取 `LegacyScore.Score`。

**Mods。** `Mod` 解析支持整数、字符串、缩写数组，以及带有 `acronym` / `settings` 的对象数组，以适配不同接口返回的数据格式。

## API 与文档

| 入口 | 主要用途 |
|---|---|
| [OsuClient](src/MintOsuApi/OsuClient.cs) | v2：用户、成绩、谱面、排名、多人比赛、新闻、论坛等 |
| [OsuV1Client](src/MintOsuApi/OsuV1Client.cs) | v1：用户、谱面、成绩、最佳成绩、最近成绩、比赛与回放 |
| [模型](src/MintOsuApi/Models) / [枚举](src/MintOsuApi/Enums) | v2 返回数据和请求参数类型 |
| [契约验证](tools/MintOsuApi.ContractValidation/README.md) | 真实响应采集、离线回放和字段兼容性检查 |

方法签名和 XML 注释可在 IDE 中查看。是否能调用某个接口取决于令牌的授权方式和范围。

## 本地开发

库与测试目标框架为 .NET 8，契约验证工具为 .NET 9。构建整个解决方案需要 .NET 9 SDK 或更高版本，并安装 .NET 8 运行时以运行测试。

在仓库根目录执行：

```sh
dotnet build MintOsuApi.sln
dotnet test MintOsuApi.sln
dotnet pack src/MintOsuApi/MintOsuApi.csproj -c Release -o artifacts
```

常规测试使用本地 JSON 样本和模拟 HTTP 响应，无需 osu! 凭据。真实 API 契约验证需单独运行，具体流程见上面的工具文档。

## 反馈与贡献

欢迎通过 [Issues](https://github.com/Tsurumaki-Kokoro/mint-osuapi/issues) 报告问题或提出功能建议，也欢迎提交 Pull Request。

报告问题时，请提供调用的方法、复现步骤、预期和实际结果，以及 .NET 和库版本。若问题涉及响应解析，可附上去除个人信息的最小 JSON 样本；请勿提交密钥或访问令牌。

修改接口或模型时，请补充相应测试，并保持中英文 README 内容一致。

## 许可证

采用 MIT 许可证，详见 [LICENSE](LICENSE)。
