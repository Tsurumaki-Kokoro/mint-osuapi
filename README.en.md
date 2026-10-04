# MintOsuAPI

[简体中文](README.md) | **English**

[![.NET 8+](https://img.shields.io/badge/.NET-8%2B-512BD4)](src/MintOsuApi/MintOsuApi.csproj) [![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
MintOsuAPI is a .NET client for the osu! API, providing strongly typed data access for bots, desktop applications, and backend services. Extracted from MintAPI, it can be used independently.

## Features

- **v2 and v1** — OAuth2 v2 API and the legacy API-key-based v1 API.
- **Typed models** — Users, scores, beatmaps, rankings, multiplayer matches, and related enums and responses.
- **Automatic authentication** — Public access-token acquisition, disk caching, and one refresh after a 401 response.
- **Async calls** — `async` / `await` with `CancellationToken` support.
- **Payload compatibility** — Modern and legacy score fields, multiple Mods formats, and polymorphic responses.
- **Custom HTTP** — Accepts a pre-configured `HttpClient`.

Targets **.NET 8** and supports .NET 8 or later. The library does not depend on MintAPI, a database, Redis, or a browser.

## Installation

Install through NuGet:

```sh
dotnet add package MintOsuAPI
```

For local development, reference the source project from your consuming project directory:

```sh
dotnet add reference /path/to/mint-osuapi/src/MintOsuApi/MintOsuApi.csproj
```

The package name is `MintOsuAPI`; the code namespace is `MintOsuApi`.

## Quick start

Create an OAuth application in your osu! account settings to obtain a client ID and secret. This example reads credentials from environment variables:

### API v2 (OAuth2 client credentials)

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

`OsuClient` handles the token exchange for you. Access tokens are cached on
disk so a fresh token is only requested when the cached one is near expiry.
The default location is `%APPDATA%\mintosuapi\tokens` on Windows and is resolved
through .NET’s `ApplicationData` directory elsewhere (typically
`~/.config/mintosuapi/tokens`); pass `tokenDirectory` to override it.
The cache key is a hash of the client ID, secret and scopes rather than the
path, so existing tokens stay valid if you move the directory.

Do not share a `TokenStore` directory between applications that use different
credentials — use one directory per credential set.

### Bringing your own HttpClient

For dependency injection, pass a pre-configured `HttpClient`. This mirrors what
the convenience constructor builds:

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

This constructor sets the `x-api-version` header but leaves `User-Agent` to
you; osu! expects a descriptive one. Disposing `OsuClient` also disposes the
supplied `HttpClient`, so account for that when managing its lifetime.

The built-in authentication handler uses the `public` scope with the
client-credentials grant. Endpoints requiring user authorization (such as chat,
forum writes, friends, and `GetMeAsync`) need an appropriately authorized
`HttpClient`; the default constructor does not perform an authorization-code flow.

### API v1 (legacy, API key)

```csharp
using MintOsuApi.V1;

string apiKey = Environment.GetEnvironmentVariable("OSU_API_KEY")!;

using var client = new OsuV1Client(apiKey);

V1User? user = await client.GetUserAsync("peppy");
List<V1Score> best = await client.GetUserBestAsync("peppy", limit: 50);
string replayData = await client.GetReplayAsync(beatmapId: 75, user: "peppy");
```

## Error handling

| Exception | Thrown by | Meaning |
|---|---|---|
| `OsuApiException` | `OsuClient` | v2 returned an error payload |
| `OsuApiV1Exception` | `OsuV1Client` | v1 returned an error, or the API key was rejected |
| `V1ReplayUnavailableException` | `OsuV1Client.GetReplayAsync` | the score has no replay available |

Non-success HTTP responses from v2 throw `HttpRequestException` before API error
payload parsing.

The API-specific exceptions live in the `MintOsuApi` and `MintOsuApi.V1` namespaces.

## Compatibility notes

**Legacy scores in multiplayer matches.** `MintOsuApi.Models.MatchGame.Scores` is a
`List<LegacyScore>`. A dedicated converter accepts both the old `score` field
and the newer `total_score`, preferring `total_score` when both are present.
If you accessed this property on an earlier version, read `LegacyScore.Score`
rather than `Score.TotalScore`.

**Mods.** `Mod` parsing accepts integers, strings, and both acronym arrays and
object arrays carrying `acronym`/`settings`, matching what the live API returns
across endpoints.

## API and documentation

| Resource | Purpose |
|---|---|
| [OsuClient](src/MintOsuApi/OsuClient.cs) | v2: users, scores, beatmaps, rankings, multiplayer matches, news, forums, and more |
| [OsuV1Client](src/MintOsuApi/OsuV1Client.cs) | v1: users, beatmaps, scores, best scores, recent scores, matches, and replays |
| [Models](src/MintOsuApi/Models) / [Enums](src/MintOsuApi/Enums) | v2 response models and request parameter types |
| [Contract validation](tools/MintOsuApi.ContractValidation/README.md) | Live-response capture, offline replay, and field compatibility checks |

Method signatures and XML comments are available in your IDE. Access to an endpoint depends on the token grant and scopes.

## Development

The library and tests target .NET 8; the contract validation tool targets .NET 9. Building the full solution requires the .NET 9 SDK or later, plus the .NET 8 runtime to run tests.

Run from the repository root:

```sh
dotnet build MintOsuApi.sln
dotnet test MintOsuApi.sln
dotnet pack src/MintOsuApi/MintOsuApi.csproj -c Release -o artifacts
```

Regular tests use local JSON fixtures and mocked HTTP responses; no osu! credentials are needed. Live API contract validation is opt-in and described in the tool documentation linked above.

## Feedback and contributing

Report bugs and request features through [Issues](https://github.com/Tsurumaki-Kokoro/mint-osuapi/issues). Pull requests are welcome.

Include the method called, reproduction steps, expected and actual results, and your .NET and library versions. For parsing issues, attach a minimal JSON sample with personal information removed. Do not include secrets or access tokens.

When changing endpoints or models, include relevant tests and keep the Chinese and English READMEs aligned.

## License

MIT — see [LICENSE](LICENSE).
