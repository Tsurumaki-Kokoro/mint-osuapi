# NuGet 发布 / Publishing to NuGet

## 中文

工作流 [Publish to NuGet](../.github/workflows/publish-nuget.yml) 仅支持手动触发。填写版本号后，依次构建、测试、打包并直接上传至 NuGet.org。包只保存在运行器临时目录，不上传到 GitHub Actions artifacts。

### 首次配置

1. 在 GitHub 仓库的 **Settings → Secrets and variables → Actions → Variables** 中添加 `NUGET_USER`，值为 NuGet.org 的个人资料用户名（不是邮箱）。
2. 登录 NuGet.org，在 **Trusted Publishing** 中创建 GitHub 发布策略：
   - Repository Owner：`Tsurumaki-Kokoro`
   - Repository：`mint-osuapi`
   - Workflow File：`publish-nuget.yml`
   - Environment：留空
   - 发布权限：允许发布 `MintOsuAPI`；首次发布时需要允许创建新包。

工作流使用 OIDC 获取短期发布密钥，无需在 GitHub 保存长期 NuGet API Key。详见 [NuGet 官方说明](https://learn.microsoft.com/en-us/nuget/nuget-org/trusted-publishing)。

### 发布版本

在 GitHub 的 **Actions → Publish to NuGet → Run workflow** 中选择要发布的分支，填写 `version`，例如 `0.1.0` 或 `0.2.0-beta.1`，然后运行。

版本必须为三段数字，可带预发布后缀；不要添加 `v` 前缀或 `+` 构建元数据。输入会覆盖本次构建的版本，不修改 `.csproj`，也不会创建 Git 标签或 GitHub Release。测试失败会阻止上传；已存在的版本会导致发布失败，不会静默跳过。

上传成功后，NuGet.org 仍需完成验证与索引。后续发布请使用新的版本号。

## English

The [Publish to NuGet](../.github/workflows/publish-nuget.yml) workflow runs only on manual dispatch. It builds, tests, packs, and pushes directly to NuGet.org using the version you enter. Packages remain in the runner's temporary directory and are never uploaded as GitHub Actions artifacts.

### One-time setup

1. Add the repository variable `NUGET_USER` under **Settings → Secrets and variables → Actions → Variables**. Use your NuGet.org profile name, not your email address.
2. On NuGet.org, create a GitHub **Trusted Publishing** policy with:
   - Repository Owner: `Tsurumaki-Kokoro`
   - Repository: `mint-osuapi`
   - Workflow File: `publish-nuget.yml`
   - Environment: leave empty
   - Publishing permissions: allow `MintOsuAPI`, including new package creation for the first release.

The workflow exchanges an OIDC token for a short-lived API key; no long-lived NuGet API key is stored in GitHub. See the [official NuGet guide](https://learn.microsoft.com/en-us/nuget/nuget-org/trusted-publishing).

### Publish a version

Open **Actions → Publish to NuGet → Run workflow**, select the branch to publish, enter `version` (for example `0.1.0` or `0.2.0-beta.1`), and run the workflow.

Use three numeric components with an optional prerelease suffix, without a `v` prefix or `+` build metadata. The input overrides the build version without editing the project file, creating a Git tag, or creating a GitHub Release. Failed tests prevent publishing. Existing package versions fail the push rather than being silently skipped.

NuGet.org validates and indexes the package after upload. Use a new version number for subsequent releases.
