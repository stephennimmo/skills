# skills

Personal [Claude Code](https://claude.com/claude-code) skills for Java, Quarkus and Angular development. Each folder is one skill, with a `SKILL.md` that Claude loads when the task matches the skill's description.

## Getting Started

Clone the repository and symlink each skill into `~/.claude/skills/` so Claude Code loads it in every project:

```shell
git clone git@github.com:stephennimmo/skills.git ~/projects/github/stephennimmo/skills
cd ~/projects/github/stephennimmo/skills
mkdir -p ~/.claude/skills
for skill in */; do
  ln -sfn "$PWD/${skill%/}" ~/.claude/skills/"${skill%/}"
done
```

Because the skills are symlinked, edits in this repository take effect in the next Claude Code session without reinstalling. Rerun the loop after adding a skill; `-sfn` makes it safe to run again.

## Skills

| Skill            | Covers                                                                                                                                                  |
| ---              | ---                                                                                                                                                     |
| `angular`        | Angular conventions: project creation, ng-bootstrap, signals, `inject()`, folder and naming structure, OIDC                                             |
| `java`           | Java conventions: latest LTS, Maven, records as DTOs, `Optional` returns, constructor injection, Hibernate Validator                                    |
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
```

- `java` applies to all Java code.
- `quarkus` builds on `java` and owns project-wide rules: configuration, persistence, testing setup and Containerfiles.
- `quarkus-rest` builds on `java` and `quarkus` and owns the REST layering, security and endpoint tests.
- `angular` stands on its own for any Angular code.
- `quarkus-quinoa` combines `quarkus`, `quarkus-rest` and `angular` for a Quarkus project that serves an Angular UI.

### Layout

```
skills/
├── angular/
│   └── SKILL.md
├── java/
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
2. Add a `SKILL.md` with `name` and `description` frontmatter. Write the description so it says when the skill should be used, since Claude relies on it to decide when to load the skill.
3. Symlink the folder into `~/.claude/skills/` and add it to the Skills table above.
4. Keep each rule in exactly one skill. When skills build on each other, reference the other skill by name (for example, `quarkus-rest` says to follow `java` and `quarkus`) instead of copying its rules.
