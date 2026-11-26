# Team Standards

Source-controlled coding standards shared across team projects. Claude Code
loads them through `@` imports in each project's `CLAUDE.md`.

## Layout

```
team-standards/
├── index.md                    # Entry point that projects import
└── dotnet/
    ├── csharp-code-style.md    # C# code style rules
    ├── unit-testing.md         # Unit testing standards
    └── structure.md            # Project and solution structure
```

## Using these standards in a project

Clone this repo as a **sibling** of your project folder:

```
Projects/
├── team-standards/   <- this repo
└── MyProject/
    └── CLAUDE.md
```

Then add one line to the project's `CLAUDE.md`:

```markdown
@../team-standards/index.md
```

The first time Claude Code sees the external import, it asks you to approve it.

## Contributing

- Make changes through pull requests so the team can review them.
- Keep the docs short. Everything imported is loaded into every Claude Code
  session, so write rules rather than tutorials.
- Write rules as instructions ("Use X", "Don't do Y") and give a reason when
  the rule isn't obvious.
