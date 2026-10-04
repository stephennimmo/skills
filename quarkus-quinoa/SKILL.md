---
name: quarkus-quinoa
description: Full-stack Quarkus + Angular projects using Quinoa. Use when creating a Quarkus project with an Angular UI, adding an Angular UI to an existing Quarkus project, or changing how the UI is built and served by Quarkus.
---

Follow the `quarkus` skill for project conventions (configuration, dev services, persistence, Containerfiles).
Follow the `quarkus-rest` skill for the REST API the UI calls.
Follow the `angular` skill for the Angular code itself.

## Adding the UI

The Angular project always lives in `src/main/webui`.

```shell
./mvnw quarkus:add-extension -Dextensions="io.quarkiverse.quinoa:quarkus-quinoa"
NG_PROJECT_NAME=crm-ng
cd src/main
ng new --directory webui --skip-git --routing --style scss --ssr false --zoneless false --defaults $NG_PROJECT_NAME
cd webui
ng add @ng-bootstrap/ng-bootstrap --skip-confirmation
ng generate environments
cd ../../..
yq -i '.quarkus.quinoa.enable-spa-routing = true' src/main/resources/application.yaml
yq -i '.quarkus.quinoa.build-dir = "dist/'$NG_PROJECT_NAME'/browser"' src/main/resources/application.yaml
```

- This is the `angular` skill's project setup, with `--skip-git` because the Quarkus project already owns the repository.
- The two `yq` commands are the only Quinoa configuration needed.

## Running

- Quarkus serves both the API and the UI on port 8080. In dev mode, Quinoa starts the Angular dev server (port 4200) and proxies to it.
- The API stays under `/api/v1/...`, so it never collides with Angular routes.

## Security

- The UI and the API use the same OIDC provider. The UI signs in with a public OIDC client (no client secret) and sends the access token as a bearer token to the API, which checks roles with `@RolesAllowed` (see the `quarkus-rest` skill).
