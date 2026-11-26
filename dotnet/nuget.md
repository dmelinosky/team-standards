# NuGet Guidance (.NET)

Package versions are managed centrally in `Directory.Packages.props` (see structure.md).

## Updating packages
- Update packages with [dotnet-outdated](https://github.com/dotnet-outdated/dotnet-outdated) (`dotnet-outdated-tool`).
- Install it as a local tool in the repo's tool manifest, so everyone runs the same version:
  ```sh
  dotnet new tool-manifest   # only if .config/dotnet-tools.json doesn't exist yet
  dotnet tool install dotnet-outdated-tool
  ```
  Commit `.config/dotnet-tools.json`. After cloning, run `dotnet tool restore`.
- Common commands, run from the repo root:
  - `dotnet outdated` lists outdated packages.
  - `dotnet outdated -u:prompt` asks before each update.
  - `dotnet outdated -vl Major -u` updates within the current major version only.
  - `dotnet outdated -u` updates everything to the latest stable version.
- Check that the updates land in `Directory.Packages.props`, not as `Version` attributes in `.csproj` files.
- Treat major-version updates as their own change. Read the release notes and update them in a separate pull request from minor and patch updates.
- After updating, run `dotnet build` and `dotnet test`. Don't merge updates that break the build or tests.
