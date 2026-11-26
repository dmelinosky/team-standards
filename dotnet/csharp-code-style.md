# C# Code Style Rules

## StyleCop comes first
- StyleCop is the primary code style authority. Where a rule in this document conflicts with StyleCop, follow StyleCop.
- Every project references `StyleCop.Analyzers`. Add it once in `Directory.Build.props` so no project is missed:
  ```xml
  <ItemGroup>
    <PackageReference Include="StyleCop.Analyzers" PrivateAssets="all" />
  </ItemGroup>
  ```
  Set the version in `Directory.Packages.props` (see structure.md).
- Configure StyleCop with a `stylecop.json` at the repo root, added to every project in `Directory.Build.props`:
  ```xml
  <AdditionalFiles Include="$(MSBuildThisFileDirectory)stylecop.json" Link="stylecop.json" />
  ```
- Fix StyleCop warnings rather than suppressing them. Disable a rule in `.editorconfig` only with team agreement, and give the reason in a comment.

## File headers
- Every `.cs` file starts with a file header, followed by a blank line (StyleCop SA1633).
- Use a plain comment header that contains only the file name. It needs no company name, so it works for personal projects too. Set this in `stylecop.json`:
  ```json
  {
    "$schema": "https://raw.githubusercontent.com/DotNetAnalyzers/StyleCopAnalyzers/master/StyleCop.Analyzers/StyleCop.Analyzers/Settings/stylecop.schema.json",
    "settings": {
      "documentationRules": {
        "xmlHeader": false,
        "copyrightText": "{fileName}"
      }
    }
  }
  ```
  Each file then starts like this:
  ```csharp
  // OrderService.cs

  namespace Company.Product.Feature;
  ```
- `"xmlHeader": false` means StyleCop expects a plain `//` comment, not the `<copyright company="...">` XML form, so its company-name rules (SA1640, SA1641) don't apply. SA1636 still checks that the header matches, so keep it in step when you rename a file.
- If a project needs a real copyright or license notice, change `copyrightText` in that repo's `stylecop.json` (for example `"Copyright (c) Contoso. Licensed under the MIT License."`). Don't turn SA1633 off.

## .editorconfig
- Every repo has an `.editorconfig` at the root. If a project doesn't have one, add it. If it has one, add any of the settings below that are missing.
- StyleCop enforces the code style, so don't repeat its rules in `.editorconfig` (braces, `String` vs `string`, using order, field and constant naming). The settings below are the ones StyleCop doesn't cover, or that stop IDE fixes and `dotnet format` from producing code StyleCop rejects.
- Don't add a `:warning` severity to settings that StyleCop already enforces. It only adds a second squiggle for the same problem.

```ini
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true
indent_style = space

[*.{csproj,props,targets,slnx,json,xml}]
indent_size = 2

[*.cs]
indent_size = 4
max_line_length = 120

# StyleCop enforces the code style. The settings below cover what it doesn't, or tell
# IDE fixes and `dotnet format` to produce code that StyleCop accepts.

# Stops IDE fixes and `dotnet format` moving usings outside the namespace (StyleCop SA1200).
csharp_using_directive_placement = inside_namespace

# Stops IDE code cleanup removing `this.` (StyleCop SA1101).
dotnet_style_qualification_for_field = true
dotnet_style_qualification_for_property = true
dotnet_style_qualification_for_method = true
dotnet_style_qualification_for_event = true

# File-scoped namespaces. StyleCop has no rule for this.
csharp_style_namespace_declarations = file_scoped:warning

# var when the type is obvious. StyleCop has no rule for this.
csharp_style_var_when_type_is_apparent = true:suggestion
csharp_style_var_for_built_in_types = true:suggestion
csharp_style_var_elsewhere = false:suggestion
```

- Put rule suppressions that the team has agreed to (see "StyleCop comes first") at the end of the `[*.cs]` section, each with a comment giving the reason.

## Naming
| Element | Convention | Example |
|---|---|---|
| Namespaces, types, methods, properties, events | PascalCase | `OrderService`, `GetTotal()` |
| Interfaces | `I` + PascalCase | `IOrderRepository` |
| Parameters, locals | camelCase | `orderId` |
| Private fields | camelCase, no `_` prefix | `logger` |
| Constants and static readonly fields | PascalCase | `MaxRetryCount` |
| Async methods | `Async` suffix | `LoadOrderAsync()` |
| Generic type parameters | `T` or `T` + descriptive name | `TResult` |

- Use full words, not abbreviations (`customer`, not `cust`). Well-known acronyms are fine: two letters stay uppercase (`IO`), longer ones use PascalCase (`Http`, `Json`).
- Name booleans as questions: `isEnabled`, `hasItems`, `canRetry`.
- Prefix every access to an instance field, property or method with `this.`: `this.logger`, `this.Total`, `this.LoadOrderAsync()`. Fields have no `_` prefix, so `this.` is what tells members apart from locals and parameters (StyleCop SA1101).

## File and type layout
- Put one top-level type in each file, and name the file after the type.
- Use file-scoped namespaces: `namespace Company.Product.Feature;`.
- Make namespaces match the folder structure.
- Order members as follows: constants, fields, constructors, properties, public methods, then private methods.
- Put `using` directives inside the namespace (StyleCop SA1200), with `System.*` first. Remove unused usings. With a file-scoped namespace, the usings go directly after the `namespace` line:
  ```csharp
  namespace Company.Product.Feature;

  using System.Text.Json;
  using Company.Product.Core;
  ```
- The `.editorconfig` setting `csharp_using_directive_placement = inside_namespace` stops IDE fixes and `dotnet format` moving usings back outside (see .editorconfig above).

## Formatting
- Indent with 4 spaces. Use Allman braces, with each brace on its own line.
- Always use braces, even for single-line `if`/`else`/`for`/`while` bodies.
- Aim for lines under 120 characters.
- Use `var` when the type is obvious from the right-hand side. Otherwise use the explicit type.
- Use the language keywords (`string`, `int`) rather than the framework types (`String`, `Int32`).
- Run `dotnet format` before committing. The `.editorconfig` is the source of truth for anything it covers.

## Modern C# usage
- Prefer `record` / `record struct` for immutable data carriers and DTOs.
- Use primary constructors for simple dependency injection. Don't use them when you need validation or a mutable backing field.
- Use pattern matching (`is null`, `is not null`, switch expressions) instead of chains of type checks.
- Use collection expressions (`[]`, `[..items]`) where they work.
- Use `required` and `init` members instead of large constructors for configuration-style objects.
- Mark classes `sealed` unless they're designed for inheritance.

## Async
- Go async all the way down. Never block on async code with `.Result`, `.Wait()` or `.GetAwaiter().GetResult()`.
- Accept a `CancellationToken` as the last parameter on public async methods, and pass it through.
- Don't use `async void`, except in event handlers.
- Return `Task` / `ValueTask` directly when a method only forwards a call and has no `using` or `try`.

## Error handling
- Throw specific exceptions (`ArgumentException`, `InvalidOperationException`, or a domain exception). Never throw `Exception`.
- Validate arguments with the guard helpers: `ArgumentNullException.ThrowIfNull(x)`, `ArgumentException.ThrowIfNullOrWhiteSpace(s)`.
- Don't catch `Exception` unless you're at an application boundary (middleware, a top-level handler), and log it there.
- Never swallow exceptions silently. Rethrow with `throw;`, not `throw ex;`.

## Dependencies and design
- Use constructor injection through `Microsoft.Extensions.DependencyInjection`. Don't use service locators.
- Depend on interfaces at architectural boundaries (I/O, external services, time). Don't create interfaces for plain classes that have only one implementation and no seam.
- Inject `TimeProvider` instead of calling `DateTime.Now` / `DateTime.UtcNow`, so the code can be tested.
- Bind configuration with `IOptions<T>` and validate it on startup.

## Logging
- Use `ILogger<T>` with message templates, not interpolated strings: `this.logger.LogInformation("Loaded order {OrderId}", orderId);`.
- On hot paths, prefer the `[LoggerMessage]` source generator.
- Never log secrets, tokens or personal data.

## Comments and documentation
- Put XML doc comments (`///`) on public APIs of shared libraries.
- Use comments to explain *why*, not *what*. Delete commented-out code instead of committing it.
- Use `// TODO:` only with a ticket reference: `// TODO(ABC-123): ...`.
