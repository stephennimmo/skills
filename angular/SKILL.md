---
name: angular
description: Angular coding conventions. Use when creating an Angular project, or when writing, generating or reviewing any Angular code, including pages, shared components, modals, guards, services, model interfaces and OIDC login.
---

## Creating a Project

```shell
NG_PROJECT_NAME=crm-ng
ng new --routing --style scss --ssr false --zoneless false --defaults $NG_PROJECT_NAME
cd "$NG_PROJECT_NAME"
ng add @ng-bootstrap/ng-bootstrap --skip-confirmation
ng generate environments
git add .
git commit -m 'Angular project init'
```

- Use the latest LTS release of Angular.
- The dev server runs on port 4200.
- Inside an existing Quarkus project using Quinoa, add `--skip-git` to `ng new` and skip the `git` commands. Follow the `quarkus-quinoa` skill for where the project goes and how Quarkus serves it.

## Coding Conventions

- Use ng-bootstrap components wherever possible: https://ng-bootstrap.github.io/#/components
- Use signals.
- Use `inject()` rather than constructor injection. (This is the opposite of the Java convention.)
- In a model interface, prefix the ID with the domain object name, never just `id`:
```typescript
export interface Account {
  accountId: number;
  name: string;
}
```

## Project Structure

| Kind             | Folder                 | File suffix     | Class name suffix |
| :---             | :---                   | :---            | :---              |
| Page component   | `src/app/pages`        | `.page.ts`      | `Page`            |
| Shared component | `src/app/pages/shared` | `.component.ts` | `Component`       |
| Guard            | `src/app/guards`       | `.guard.ts`     | `Guard`           |
| Service          | `src/app/services`     | `.service.ts`   | `Service`         |

- A modal component used by only one page goes in that page's folder.
- Shared components are ones used across pages, such as the navbar.

## Security

- Authentication uses OIDC. Keycloak is the reference provider, but stay provider-agnostic: only standard OIDC configuration, so any OIDC provider can be swapped in.
- Never commit secrets. A browser app uses a public OIDC client, so it never needs a client secret.
