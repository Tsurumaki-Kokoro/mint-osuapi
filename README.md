# MintOsuAPI

An osu! API client for .NET. Covers the v2 API (OAuth2, 59 methods) and the
legacy v1 API (API key, 7 methods).

Targets `net8.0`, so it can be referenced from .NET 8 and any later runtime.

## Install

```
dotnet add package MintOsuAPI
```

## API v2 (OAuth2 client credentials)

```csharp
using MintOsuApi;
using MintOsuApi.Enums;

using var client = new OsuClient(clientId, clientSecret);

User user = await client.GetUserAsync("peppy", GameMode.Osu);
Beatmap map = await client.GetBeatmapAsync(beatmapId: 75);
List<Score> scores = await client.GetUserScoresAsync(
    user.Id, ScoreType.Best, mode: GameMode.Osu, limit: 10);
```

`OsuClient` handles the token exchange for you. Access tokens are cached on
disk so a fresh token is only requested when the cached one is near expiry.
The default location is `%APPDATA%\mintosuapi\tokens` on Windows and
`~/.config/mintosuapi/tokens` elsewhere; pass `tokenDirectory` to override it.
The cache key is a hash of the client ID, secret and scopes rather than the
path, so existing tokens stay valid if you move the directory.

Do not share a `TokenStore` directory between applications that use different
credentials — use one directory per credential set.

### Bringing your own HttpClient

For dependency injection, pass a pre-configured `HttpClient`. This mirrors what
the convenience constructor builds:

```csharp
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
you; osu! expects a descriptive one.

## API v1 (legacy, API key)

```csharp
using MintOsuApi.V1;

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

All three live in the `MintOsuApi` and `MintOsuApi.V1` namespaces.

## Compatibility notes

**Legacy scores in multiplayer matches.** `MintOsuApi.MatchGame.Scores` is a
`List<LegacyScore>`. A dedicated converter accepts both the old `score` field
and the newer `total_score`, preferring `total_score` when both are present.
If you accessed this property on an earlier version, read `LegacyScore.Score`
rather than `Score.TotalScore`.

**Mods.** `Mod` parsing accepts integers, strings, and both acronym arrays and
object arrays carrying `acronym`/`settings`, matching what the live API returns
across endpoints.

## License

MIT — see [LICENSE](LICENSE).
