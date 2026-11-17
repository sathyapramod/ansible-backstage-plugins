# Scaffolder Backend Split — Implementation Guide

> **Architecture reference:** [Content Experience Architecture](https://github.com/ansible/ansible-rhdh-plugins/blob/portal-plugin-research/.sdlc/research/plugin-factory/Content%20Experience%20Architecture.md) (private repo — §2.2 covers workspace layout, §7.3–§7.4 covers backend boundary)
>
> **Scope:** Split the monolithic `scaffolder-backend-module-backstage-rhaap` into two backend
> modules: one for AAP integration and one for Portal scaffolder actions. Phase 1 moves EE action
> calls to an in-process lifecycle service while retaining the legacy HTTP routes for compatibility.

---

## 1. Current State: What Lives in the Monolith Today

The scaffolder backend module today is a single package that registers AAP Controller actions, EE definition authoring actions, publish actions, template filters, and an autocomplete provider — all under one `moduleId: 'ansible'`. These concerns belong to different domains and should be independently deployable.

### 1.1 Repository layout (main branch)

```
plugins/
├── backstage-rhaap/                          # AAP frontend plugin
├── backstage-rhaap-common/                   # Shared: ansibleServiceRef, AAPClient, ScmClientFactory
├── catalog-backend-module-rhaap/             # Catalog providers (PAH, git, AAP, EE)
├── auth-backend-module-rhaap-provider/       # AAP auth
├── scaffolder-backend-module-backstage-rhaap/ # ← monolithic, being split
└── self-service/                             # Frontend plugin
packages/
└── backend/src/index.ts                      # Backend wiring
```

### 1.2 Module registration and backend wiring

`plugins/scaffolder-backend-module-backstage-rhaap/src/module.ts` — a single `createBackendModule({ pluginId: 'scaffolder', moduleId: 'ansible' })` that registers everything.

`packages/backend/src/index.ts` has one entry:

```ts
backend.add(
  import('@ansible/plugin-scaffolder-backend-module-backstage-rhaap'),
);
```

### 1.3 Config schema (single `config.d.ts`)

| Config path               | Belongs to               |
| ------------------------- | ------------------------ |
| `ansible.rhaap`           | AAP module               |
| `ansible.creatorService`  | Portal-scaffolder module |
| `ansible.devSpaces`       | Portal-scaffolder module |
| `catalog.providers.rhaap` | AAP module               |

Backstage validates against the union of installed schemas — a mismatch is a startup failure.

### 1.4 String contract between autocomplete and EE definition

`autocomplete/utils.ts:formatSource()` emits display strings (`"Private Automation Hub / <repo>"`, `"<Provider> / <host> / <org> / <repo>"`) that `createEEDefinition.ts` parses back into structured data. The split puts a package boundary between producer and consumer. This format must be preserved as a tested contract.

---

## 2. Target State

Per the Content Experience Architecture §2.2, the end state separates AAP integration from content-development ownership. Phase 1 uses the Portal scaffolder module as an intermediate implementation; the content-development move remains a follow-on.

```
workspaces/
├── aap/plugins/
│   ├── aap-common/  aap-node/  aap/  aap-backend/     # renamed from backstage-rhaap-*
│   ├── auth-backend-module-aap-provider/
│   ├── catalog-backend-module-aap/
│   └── scaffolder-backend-module-aap/                   # NEW — AAP scaffolder actions
│
└── content-development/plugins/
    ├── content-development-common/                       # Creator/SCM and EE lifecycle contracts
    ├── catalog-backend-module-content-development/       # Content-owned EE catalog provider
    └── scaffolder-backend-module-content-development/    # Creator CLI / ADT actions + autocomplete
```

**Phase boundary.** PRs #993–#999 create `portal-scaffolder-backend-module`, which owns the moved content actions and lifecycle adapter in Phase 1. A follow-on moves that code and the related shared/catalog packages to `content-development`; this guide treats that workspace as the end-state source of truth.

### Module identity

| Before                             | After                                                                                                          |
| ---------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| `moduleId: 'ansible'` (one module) | Phase 1: `aap` + `portal-scaffolder`; end state: `aap` + `content-development` (same `pluginId: 'scaffolder'`) |

**Breaking migration.** Action IDs and template filter names remain stable, but the monolithic package is removed. Operators load the AAP and Portal scaffolder modules, then migrate `aap-api-cloud` references to `aap-api` for AAP resources or `portal-catalog` for collections. No compatibility shim is retained for the removed scaffolder package.

---

## 3. What Moves, What Stays, and What Remains Transitional

### 3.1 Actions, filters, and autocomplete

| Action ID                            | File                              | Target module     |
| ------------------------------------ | --------------------------------- | ----------------- |
| `ansible:content:create`             | `actions/ansible.ts`              | portal-scaffolder |
| `ansible:create:ee-definition`       | `actions/createEEDefinition.ts`   | portal-scaffolder |
| `ansible:prepare:publish`            | `actions/prepareForPublish.ts`    | portal-scaffolder |
| `rhaap:create-project`               | `actions/aapCreateProject.ts`     | aap               |
| `rhaap:create-execution-environment` | `actions/aapCreateEEEnv.ts`       | aap               |
| `rhaap:create-job-template`          | `actions/aapCreateJobTemplate.ts` | aap               |
| `rhaap:launch-job-template`          | `actions/aapLaunchJobTemplate.ts` | aap               |
| `rhaap:clean-up`                     | `actions/aapCleanUp.ts`           | aap               |

**Template filters** → aap: `useCaseNameFilter`, `resourceFilter`, `multiResourceFilter`, `uuidFilter`

**Autocomplete providers** follow the data source: AAP Controller lookups use `id: 'aap-api'` in the AAP module; collection lookups use `id: 'portal-catalog'` in the Portal scaffolder module.

### 3.2 `backstage-rhaap-common` dependency edge and config reads

`ansible:prepare:publish` imports `ScmClientFactory` from `@ansible/backstage-rhaap-common`. That barrel also exports `AAPClient`, `AAPService`, the EE lifecycle contracts, and permissions. The Portal scaffolder module therefore retains a transitive dependency on AAP code. Full independence requires splitting this package into a shared SCM/lifecycle package and AAP-specific code; that work is outside Phase 1.

`createEEDefinition.ts` is ~1240 lines containing PAH repo identifiers, SCM provider/host/org/repo parsing, galaxy server config, and semver-based collection merging. It moves as-is in Phase 1.

**Its config dependency is wider than `ansible.creatorService` + `ansible.devSpaces`.** The file also reads:

- `ansible.rhaap.baseUrl` — to build `galaxy_server` entries for the creator service payload
- `catalog.providers.rhaap.<envId>.sync.pahCollections.repositories` — PAH repository identifiers
- `catalog.providers.rhaap.<envId>.sync.ansibleGitContents.providers.<provider>` — SCM canonical-source resolution

These are AAP/catalog config keys, not Portal scaffolder's own. The Portal action still reads them, so its `config.d.ts` declares these transitional read dependencies until the PAH/SCM resolution logic is extracted behind a content-side contract. §1.3's config table describes ownership, not every transitional read.

### 3.3 EE entity registration

Phase 1 replaces the EE action's HTTP calls with direct lifecycle-service calls. The legacy `POST /ansible/ee` and `DELETE /ansible/ee/:name` routes remain in the AAP catalog router for compatibility; the EE build route also remains.

The lifecycle uses two service references:

- `eeEntityProviderServiceRef` is root-scoped and is provided by `catalog-backend-module-rhaap`, which owns `EEEntityProvider` registration with the catalog.
- `eeEntityLifecycleServiceRef` is plugin-scoped and is provided by `portal-scaffolder-backend-module` through `EEEntityLifecycleAdapter`, which coordinates catalog permission checks, catalog operations, and the provider service.

`createEEDefinition.ts` obtains initiator credentials once and passes them to unregister and register operations. Registration requires catalog create permission; replacement and unregister require delete permission for an existing entity. The adapter skips cleanup for provider-managed entities only after authorization. It uses service credentials only when it needs to inspect entities at a shared catalog location.

### 3.4 Autocomplete providers follow the queried system

`handleAutocompleteRequest` currently branches on `resource === 'collections'` _before_ any Controller call — it queries the Backstage catalog via `getCollections()`, not AAP's Controller API. Portal's `CollectionsPicker` field currently reaches this branch through the same `aap-api-cloud` provider used by every other AAP resource.

The split gives collections its own provider rather than retaining that coupling:

- `scaffolder-backend-module-aap` registers `aap-api` for Controller-backed AAP resources.
- `portal-scaffolder-backend-module` registers `portal-catalog` for `collections`.
- `getCollections()` and `buildCollectionsFromCatalogEntities()` live with the Portal provider. They use auth, discovery, and logger, but do not call the Controller.
- `CollectionsPickerExtension.tsx` uses `portal-catalog`; AAP picker and template references use `aap-api`.

The Portal provider caches collection results for 60 seconds by filter and deduplicates in-flight requests for that same filter. Its catalog request uses the module's service credentials.

CEA §7.5 still describes the longer-term design: a registered content operation with a declared schema and permission (e.g. `content.collections.list`), called through a typed client rather than a direct provider lookup, served by a dedicated `scaffolder-backend-module-content-operations` module (CEA §2.2). The separate `portal-catalog` provider is the Phase 1 step before that work.

---

## 4. New Packages and Compatibility Package

### 4.1 `scaffolder-backend-module-aap`

Located at `workspaces/aap/plugins/scaffolder-backend-module-aap/`.

- Actions: `rhaap:create-project`, `rhaap:create-execution-environment`, `rhaap:create-job-template`, `rhaap:launch-job-template`, `rhaap:clean-up`
- Autocomplete: `aap-api` for AAP resources
- Filters: `useCaseNameFilter`, `resourceFilter`, `multiResourceFilter`, `uuidFilter`
- Config: `ansible.rhaap`, `catalog.providers.rhaap`
- `moduleId: 'aap'`

### 4.2 `portal-scaffolder-backend-module`

Located at `workspaces/portal-scaffolder/plugins/portal-scaffolder-backend-module/`.

- Actions: `ansible:content:create`, `ansible:create:ee-definition`, `ansible:prepare:publish`
- Autocomplete: `portal-catalog` for collections
- Config: `ansible.creatorService`, `ansible.devSpaces`
- `moduleId: 'portal-scaffolder'`

The module uses the EE lifecycle contracts and provider service supplied by `@ansible/backstage-rhaap-common` and `catalog-backend-module-rhaap`.

### 4.3 Content-development end state

The follow-on replaces the Phase 1 Portal ownership with these packages under `workspaces/content-development/plugins/`:

- `content-development-common` for creator/SCM clients and EE lifecycle contracts.
- `catalog-backend-module-content-development` for the EE catalog provider and root-scoped provider service.
- `scaffolder-backend-module-content-development` for creator CLI / ADT actions, `portal-catalog`, and the lifecycle adapter.

The action IDs and autocomplete-provider IDs remain unchanged during that move. Operators replace the Phase 1 Portal module with the content-development catalog and scaffolder modules only when the follow-on ships.

### 4.4 Breaking package migration

`scaffolder-backend-module-backstage-rhaap` is removed as part of this migration. It is not retained as a compatibility shim. Consumers install the AAP and Portal scaffolder modules; this avoids maintaining two deployment paths and prevents duplicate registrations.

---

## 5. Action Plan

### Phase 0 — Contract freeze

Capture a machine-readable inventory before changing anything.

| Item                                                                               | Status |
| ---------------------------------------------------------------------------------- | ------ |
| Register all `ansible:*` and `rhaap:*` action IDs                                  | ✅     |
| Register autocomplete resources (`collections`, `collection_sources`, …)           | ✅     |
| Document provider migration from `aap-api-cloud` to `aap-api` and `portal-catalog` | ✅     |
| Capture the picker↔action source-string format as an explicit contract (§1.4)     | ✅     |

### Phase 1 PR series — Backend split and EE lifecycle migration (#993–#999)

PRs #993–#998 are merged. PR #999 contains the final backend cutover and remains pending merge.

| PRs       | Delivery status                                                                |
| --------- | ------------------------------------------------------------------------------ |
| #993–#995 | Merged: AAP actions, autocomplete, filters, package wiring, and dynamic export |
| #996      | Merged: Portal scaffolder content actions and creator-service helpers          |
| #997      | Merged: EE definition/publish actions and lifecycle-service integration        |
| #998      | Merged: `aap-api` and `portal-catalog` picker/provider migration               |
| #999      | Pending: Portal-core cutover, split-module wiring, and monolith removal        |

| Item                                                                            | Notes                                                                                                                                                                                          |
| ------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Create `workspaces/aap/plugins/scaffolder-backend-module-aap/`                  | `aap*.ts`, autocomplete, filters, `ansibleService` deps. `moduleId: 'aap'`                                                                                                                     |
| Create `workspaces/portal-scaffolder/plugins/portal-scaffolder-backend-module/` | Content actions and the `portal-catalog` collections provider; `moduleId: 'portal-scaffolder'`                                                                                                 |
| Split `config.d.ts` per module                                                  | AAP owns `ansible.rhaap` + `catalog.providers.rhaap`; Portal scaffolder owns `ansible.creatorService` + `ansible.devSpaces` and declares transitional AAP/catalog read dependencies — see §3.2 |
| Each module gets its own `src/dynamic/index.ts`                                 | One dynamic plugin installer per shippable module                                                                                                                                              |
| Update backend wiring                                                           | Register the AAP and Portal scaffolder modules                                                                                                                                                 |
| Migrate autocomplete-provider references                                        | Replace `aap-api-cloud` with `aap-api` for AAP resources and `portal-catalog` for collections, including the three `CollectionsPickerExtension.tsx` provider strings                           |
| Add the EE lifecycle service                                                    | The Portal action calls it directly for registration and cleanup; preserve caller authorization, SCM cleanup, provider unregistering, and the provider-managed entity skip                     |
| Retain legacy EE routes                                                         | Keep `POST /ansible/ee` and `DELETE /ansible/ee/:name` for compatibility while direct lifecycle calls serve the Portal action                                                                  |
| Remove original package                                                         | No compatibility shim; release migration notes cover the required package, dynamic-plugin, and provider-reference changes                                                                      |
| Split unit tests and shared `src/mock/` fixtures                                | Shared fixtures duplicated or extracted to a test-util                                                                                                                                         |

**Verification:** scaffolder module tests pass; smoke-test an EE template end-to-end in the dev app.

### Phase 2 — Remove the Portal dependency on `backstage-rhaap-common`

| Item                                   | Notes                                                                                               |
| -------------------------------------- | --------------------------------------------------------------------------------------------------- |
| Create content-development packages    | Add the common, catalog, and scaffolder packages described in §4.3.                                 |
| Move Portal scaffolder ownership       | Move the content actions, autocomplete provider, and lifecycle adapter into content-development.    |
| Extract shared SCM/lifecycle contracts | Move Portal-facing SCM and EE lifecycle contracts out of the AAP common package.                    |
| Replace PAH/SCM configuration reads    | Move repository and canonical-source resolution behind a content-development contract.              |
| Validate configuration ownership       | Remove transitional `ansible.rhaap` and `catalog.providers.rhaap` reads once that API is available. |

---

## 6. Key Constraints and Contracts

| Consumer         | Contract                                                                                                                  | Provided by                        |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------- | ---------------------------------- |
| Template authors | Action IDs and template filter names unchanged; autocomplete provider references migrate to `aap-api` or `portal-catalog` | Changelog entry and migration note |
| RHDH / operator  | AAP and Portal scaffolder modules replace the monolith                                                                    | Updated `dynamic-plugins.yaml`     |

**Semver:** internal packaging change — minor bump on affected packages as long as action IDs stay stable.

---

## 7. Backend Surface

### Phase 1 implementation

| Package                                         | Registers                                                          | Owns                                                                 |
| ----------------------------------------------- | ------------------------------------------------------------------ | -------------------------------------------------------------------- |
| `@ansible/plugin-scaffolder-backend-module-aap` | `rhaap:*` actions, template filters, `aap-api`                     | AAP actions and AAP Controller autocomplete                          |
| `@ansible/portal-scaffolder-backend-module`     | Content actions, `portal-catalog`, plugin-scoped lifecycle adapter | Creator CLI / ADT actions, autocomplete, and authorized EE lifecycle |
| `catalog-backend-module-rhaap`                  | EE provider and root-scoped provider service                       | EE entity registration and provider lifecycle                        |

### Content-development end state

| Package                                                         | Owns                                                           |
| --------------------------------------------------------------- | -------------------------------------------------------------- |
| `@ansible/content-development-common`                           | Creator/SCM clients and EE lifecycle interfaces/service refs   |
| `@ansible/catalog-backend-module-content-development`           | EE catalog provider and root-scoped provider service           |
| `@ansible/plugin-scaffolder-backend-module-content-development` | Creator CLI / ADT actions, `portal-catalog`, lifecycle adapter |

---

## 8. Open Questions

| Question                                                                                                            | Blocking      |
| ------------------------------------------------------------------------------------------------------------------- | ------------- |
| Does #999 stop at the Portal intermediate module, with the §4.3 content-development move tracked as follow-on work? | Release scope |
| Which AAP/catalog reads in `createEEDefinition.ts` become a content-development API? (§3.2)                         | Phase 2       |

---

## Appendix A: Source File Map

| Concern               | Current path                                                                                                           | Target                                                                                                             |
| --------------------- | ---------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| Module registration   | `workspaces/aap/plugins/scaffolder-backend-module-backstage-rhaap/src/module.ts`                                       | AAP scaffolder module + Portal scaffolder module                                                                   |
| AAP actions           | `workspaces/aap/plugins/scaffolder-backend-module-backstage-rhaap/src/actions/aap*.ts`                                 | `workspaces/aap/plugins/scaffolder-backend-module-aap/src/actions/`                                                |
| EE definition action  | `workspaces/aap/plugins/scaffolder-backend-module-backstage-rhaap/src/actions/createEEDefinition.ts`                   | `workspaces/portal-scaffolder/plugins/portal-scaffolder-backend-module/src/actions/`                               |
| Publish action        | `workspaces/aap/plugins/scaffolder-backend-module-backstage-rhaap/src/actions/prepareForPublish.ts`                    | `workspaces/portal-scaffolder/plugins/portal-scaffolder-backend-module/src/actions/`                               |
| Content create action | `workspaces/aap/plugins/scaffolder-backend-module-backstage-rhaap/src/actions/ansible.ts`                              | `workspaces/portal-scaffolder/plugins/portal-scaffolder-backend-module/src/actions/`                               |
| Autocomplete          | `workspaces/aap/plugins/scaffolder-backend-module-backstage-rhaap/src/autocomplete/`                                   | AAP lookups → AAP module; collection lookups → Portal scaffolder module; preserve `aap-api` / `portal-catalog` IDs |
| Template filters      | `plugins/scaffolder-backend-module-backstage-rhaap/src/filters/`                                                       | `workspaces/aap/plugins/scaffolder-backend-module-aap/src/filters/`                                                |
| AAP schemas           | `plugins/scaffolder-backend-module-backstage-rhaap/src/actions/schemas/rhaapAction*.ts`                                | `workspaces/aap/plugins/scaffolder-backend-module-aap/src/actions/schemas/`                                        |
| EE definition schema  | `workspaces/aap/plugins/scaffolder-backend-module-backstage-rhaap/src/actions/schemas/rhaapActionSchemas.ts` (partial) | `workspaces/portal-scaffolder/plugins/portal-scaffolder-backend-module/src/actions/`                               |
| Config                | `workspaces/aap/plugins/scaffolder-backend-module-backstage-rhaap/config.d.ts`                                         | Split per module                                                                                                   |
| Backend wiring        | `workspaces/portal-core/packages/backend/src/index.ts`                                                                 | Register the AAP and Portal scaffolder modules                                                                     |
| EE lifecycle contract | `workspaces/aap/plugins/backstage-rhaap-common/src/EEEntityLifecycle.ts`                                               | Retained in `backstage-rhaap-common` for Phase 1                                                                   |
| EE lifecycle adapter  | Legacy route calls in `createEEDefinition.ts`                                                                          | `workspaces/portal-scaffolder/plugins/portal-scaffolder-backend-module/src/eeEntityLifecycleAdapter.ts`            |
| EE catalog provider   | `workspaces/aap/plugins/catalog-backend-module-rhaap/src/providers/EEEntityProvider.ts`                                | Retained in `catalog-backend-module-rhaap`; the lifecycle adapter uses its provider service                        |
