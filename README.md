# skills

Personal [Claude Code](https://claude.com/claude-code) skills for Java and Quarkus development. Each folder is one skill, with a `SKILL.md` that Claude loads when the task matches the skill's description.

## Getting Started

Clone the repository and symlink each skill into `~/.claude/skills/` so Claude Code loads it in every project:

```shell
git clone git@github.com:stephennimmo/skills.git ~/projects/github/stephennimmo/skills
cd ~/projects/github/stephennimmo/skills
mkdir -p ~/.claude/skills
for skill in */SKILL.md; do
  ln -s "$PWD/$(dirname "$skill")" ~/.claude/skills/"$(dirname "$skill")"
done
```

Because the skills are symlinked, edits in this repository take effect in the next Claude Code session without reinstalling.

## Skills

| Skill              | Covers                                                                                                                                                                                         |
| ---                | ---                                                                                                                                                                                            |
| `java`             | Java conventions: latest LTS, Maven, records as DTOs, `Optional` returns, constructor injection, Hibernate Validator                                                                           |
| `quarkus-rest-api` | Quarkus REST APIs: package-by-subject Resource/Service/Repository layering, request/response records, OpenAPI, OIDC security, Panache, Flyway/PostgreSQL, configuration, tests, Containerfiles |

## Details

### Layout

```
skills/
├── java/
│   └── SKILL.md
├── quarkus-rest-api/
│   └── SKILL.md
└── README.md
```

### Adding a skill

1. Create a folder named after the skill.
2. Add a `SKILL.md` with `name` and `description` frontmatter. Write the description so it says when the skill should be used, since Claude relies on it to decide when to load the skill.
3. Symlink the folder into `~/.claude/skills/` and add it to the Skills table above.
