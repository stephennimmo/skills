# skills

Personal [Claude Code](https://claude.com/claude-code) and [Cursor Agent](https://cursor.com) skills for Java, Quarkus, Angular and MkDocs development. Each folder is one skill, with a `SKILL.md` that the agent loads when the task matches the skill's description.

## Getting Started

Clone the repository and symlink each skill into the personal skills directories for Claude Code (`~/.claude/skills/`) and the Cursor Agent (`~/.cursor/skills/`):

```shell
git clone git@github.com:stephennimmo/skills.git ~/projects/github/stephennimmo/skills
cd ~/projects/github/stephennimmo/skills
mkdir -p ~/.claude/skills ~/.cursor/skills
for skill in */; do
  ln -sfn "$PWD/${skill%/}" ~/.claude/skills/"${skill%/}"
  ln -sfn "$PWD/${skill%/}" ~/.cursor/skills/"${skill%/}"
done
```

Verify the links:

```shell
ls -l ~/.claude/skills ~/.cursor/skills
```

- Because the skills are symlinked, edits in this repository take effect in the next Claude Code or Cursor Agent session without reinstalling.
- Rerun the loop after adding a skill. `-sfn` replaces existing links, so it is safe to run again.
- Leave `~/.claude/skills/synced/` (managed by claude.ai) and `~/.cursor/skills-cursor/` (Cursor's built-in skills) alone. Skill names here must not clash with anything in those folders.

## Skills

| Skill            | Covers                                                                                                                                                  |
| ---              | ---                                                                                                                                                     |
| `angular`        | Angular conventions: project creation, ng-bootstrap, signals, `inject()`, folder and naming structure, OIDC                                             |
| `java`           | Java conventions: latest LTS, Maven, records as DTOs, `Optional` returns, constructor injection, Hibernate Validator                                    |
| `mkdocs`         | Material for MkDocs sites: `mkdocs.yaml`, `.venv`, assets layout, Red Hat branding, GitHub Pages Actions deploy, docs writing                           |
| `quarkus`        | Quarkus projects: extensions, `application.yaml` profiles, dev services, Panache entities, Flyway/PostgreSQL, testing, Protobuf/gRPC, Containerfiles    |
| `quarkus-quinoa` | Quarkus + Angular via Quinoa: adding the UI in `src/main/webui`, Quinoa config, running, shared OIDC                                                    |
| `quarkus-rest`   | Quarkus REST APIs: package-by-subject Resource/Service/Repository layering, request/response records, OpenAPI, `@RolesAllowed` security, endpoint tests |

## Details

### How the skills fit together

Skills build on each other and reference each other by name instead of repeating rules:

```
java ──► quarkus ──► quarkus-rest
              │            │
              ▼            ▼
          quarkus-quinoa ◄─┘
              ▲
              │
           angular

mkdocs   (standalone)
```

- `java` applies to all Java code.
- `quarkus` builds on `java` and owns project-wide rules: configuration, persistence, testing setup and Containerfiles.
- `quarkus-rest` builds on `java` and `quarkus` and owns the REST layering, security and endpoint tests.
- `angular` stands on its own for any Angular code.
- `quarkus-quinoa` combines `quarkus`, `quarkus-rest` and `angular` for a Quarkus project that serves an Angular UI.
- `mkdocs` stands on its own for Material for MkDocs documentation sites.

### Layout

```
skills/
├── angular/
│   └── SKILL.md
├── java/
│   └── SKILL.md
├── mkdocs/
│   └── SKILL.md
├── quarkus/
│   └── SKILL.md
├── quarkus-quinoa/
│   └── SKILL.md
├── quarkus-rest/
│   └── SKILL.md
└── README.md
```

### Adding a skill

1. Create a folder named after the skill.
2. Add a `SKILL.md` with `name` and `description` frontmatter. Write the description so it says when the skill should be used, since the agent relies on it to decide when to load the skill.
3. Rerun the symlink loop from Getting Started, and add the skill to the Skills table above.
4. Keep each rule in exactly one skill. When skills build on each other, reference the other skill by name (for example, `quarkus-rest` says to follow `java` and `quarkus`) instead of copying its rules.
