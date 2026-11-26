# Project and Solution Structure (.NET)

## Layout
```
Repo/
├── Repo.slnx
├── Directory.Build.props        # Settings for every project
├── Directory.Packages.props     # NuGet package versions for every project
├── stylecop.json                # StyleCop settings, including the file header
├── .editorconfig                # Editor and IDE settings (see csharp-code-style.md)
├── src/
│   └── Company.Product.Feature/
└── tests/
    ├── Directory.Build.props    # Settings for every test project
    └── Company.Product.Feature.Tests/
```

## Solution
- Keep one solution at the repo root, in the XML solution format (`.slnx`). Don't add `.sln` files.
- Convert an existing `.sln` with `dotnet sln migrate`, then delete the `.sln`.

## Projects
- Put production projects (apps, libraries, hosts) under `src/`.
- Put test projects (unit and integration) under `tests/`.
- Name each project folder after the project, and keep the `.csproj` directly inside it.

## Language and project settings
- Target the current LTS or STS .NET release that the solution already uses. Don't change `TargetFramework` without asking.
- Keep `<Nullable>enable</Nullable>` and `<ImplicitUsings>enable</ImplicitUsings>` on in every project.
- Treat nullable warnings as real bugs. Don't silence them with `!` unless a comment explains why the value can't be null.

## Central Package Management
- Use Central Package Management. Keep one `Directory.Packages.props` at the repo root, and set every NuGet package version there.
- Turn on transitive pinning (`CentralPackageTransitivePinningEnabled`). A `PackageVersion` entry then also sets the version of that package when it comes in as a transitive dependency, so you can raise a vulnerable transitive package by adding it to `Directory.Packages.props` instead of adding a direct reference.
- In `.csproj` and `Directory.Build.props` files, write `PackageReference` items without a `Version` attribute.
- To use a different version in one project, use `VersionOverride` on that reference only with team agreement, and give the reason in a comment.
- Add `PrivateAssets="all"` on analyzer and build-only packages (such as StyleCop) where they're referenced, not in `Directory.Packages.props`.

```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
    <CentralPackageTransitivePinningEnabled>true</CentralPackageTransitivePinningEnabled>
  </PropertyGroup>

  <ItemGroup>
    <PackageVersion Include="Microsoft.Extensions.TimeProvider.Testing" Version="x.y.z" />
    <PackageVersion Include="StyleCop.Analyzers" Version="x.y.z" />
    <PackageVersion Include="TUnit" Version="x.y.z" />
    <PackageVersion Include="TUnit.Mocks" Version="x.y.z" />
  </ItemGroup>
</Project>
```

## Shared build properties
- Don't repeat a setting in individual `.csproj` files when it applies to every project or every test project. Put it in the matching `Directory.Build.props`.

### Root `Directory.Build.props`
Applies to every project in the repo. It must set `SatelliteResourceLanguages` to `en`, so builds don't copy localized resource folders for every language from NuGet packages. It also holds the StyleCop reference and `stylecop.json` that every project needs (see csharp-code-style.md). `GenerateDocumentationFile` is on because StyleCop's documentation rules don't run without it (SA0001).

```xml
<Project>
  <PropertyGroup>
    <SatelliteResourceLanguages>en</SatelliteResourceLanguages>
    <GenerateDocumentationFile>true</GenerateDocumentationFile>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="StyleCop.Analyzers" PrivateAssets="all" />
    <AdditionalFiles Include="$(MSBuildThisFileDirectory)stylecop.json" Link="stylecop.json" />
  </ItemGroup>
</Project>
```

### `tests/Directory.Build.props`
Holds everything a test project needs: the testing package references and common test properties. A test `.csproj` should only need its `ProjectReference` to the project under test.

MSBuild only loads the nearest `Directory.Build.props`, so this file must import the root one first.

```xml
<Project>
  <Import Project="$([MSBuild]::GetPathOfFileAbove('Directory.Build.props', '$(MSBuildThisFileDirectory)../'))" />

  <PropertyGroup>
    <IsPackable>false</IsPackable>
    <!-- TUnit.Mocks needs C# 14 or later. -->
    <LangVersion>14</LangVersion>
    <!-- Test names use underscores between words (see unit-testing.md). -->
    <NoWarn>$(NoWarn);CA1707</NoWarn>
    <!-- Test classes and methods aren't XML-documented: their sentence-style names say what each test checks (see unit-testing.md). -->
    <NoWarn>$(NoWarn);SA1600;CS1591</NoWarn>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="TUnit" />
    <PackageReference Include="TUnit.Mocks" />
    <PackageReference Include="Microsoft.Extensions.TimeProvider.Testing" />
  </ItemGroup>
</Project>
```

Package versions for these references go in `Directory.Packages.props` (see Central Package Management).
