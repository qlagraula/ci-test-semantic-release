# CI/CD Pipeline Documentation

## Overview

This project uses GitHub Actions for continuous integration and deployment. The pipeline supports:

- Pull Request validation with preview deployments
- Automated releases via semantic-release
- Staging and Production deployments

## Workflow Architecture

```mermaid
flowchart TB
    subgraph PR["Pull Request Flow"]
        PR_TRIGGER["Push to PR"]
        CI["CI Workflow"]

        subgraph CI_JOBS["Checks"]
            LINT["Lint"]
            BUILD["Build"]
            TEST["Test"]
        end

        DEPLOY_PREVIEW["Deploy Preview<br/>(staging-preview)"]

        PR_TRIGGER --> CI
        CI --> CI_JOBS
        CI_JOBS --> DEPLOY_PREVIEW
    end

    PR -.->|Merge PR| MAIN_TRIGGER

    RELEASE ==> CHECK_RELEASE

    subgraph RELEASE["Release Flow"]
        MAIN_TRIGGER["Push to main"]
        RELEASE_TRIGGER["CI Workflow Completed"]
        SEMANTIC_RELEASE["Semantic Release"]
        COMMIT_ANALYSIS("Commit Analysis")


        subgraph CREATE_RELEASE["Create Release"]
            CHANGELOG_UPDATE["Changelog Update"]
            VERSION_BUMP["Version Bump"]
            GITHUB_RELEASE["Create Draft GitHub Release"]
        end

        MAIN_TRIGGER --> RELEASE_TRIGGER
        RELEASE_TRIGGER --> SEMANTIC_RELEASE
        SEMANTIC_RELEASE --> COMMIT_ANALYSIS
        COMMIT_ANALYSIS -->|Need release| CREATE_RELEASE
        COMMIT_ANALYSIS -->END_RELEASE["No release"]
    end

    subgraph STAGING["Staging Flow"]
        STAGING_MANUAL_TRIGGER["Manual dispatch"]
        CHECK_RELEASE("Has a new release been drafted?")

        CHECK_RELEASE -->|Yes| STAGING_JOBS
        CHECK_RELEASE -->|No| END["No deployment"]

        subgraph STAGING_JOBS["Checks"]
            LINT_S["Lint"]
            BUILD_S["Build"]
            TEST_S["Test"]
        end

        DEPLOY_STAGING["Deploy Staging"]
        DEPLOY_PROD_PREVIEW["Deploy Production Preview<br/>(production-preview)"]

        STAGING_MANUAL_TRIGGER --> STAGING_JOBS
        STAGING_JOBS --> DEPLOY_STAGING
        DEPLOY_STAGING --> DEPLOY_PROD_PREVIEW
    end

    subgraph PRODUCTION["Production Flow"]
        PROD_MANUAL_TRIGGER["Manual dispatch"]

        subgraph PROD_JOBS["Checks"]
            LINT_P["Lint"]
            BUILD_P["Build"]
            TEST_P["Test"]
        end

        DEPLOY_PROD["Deploy Production"]

        PROD_MANUAL_TRIGGER --> PROD_JOBS
        PROD_JOBS --> DEPLOY_PROD
    end

    GITHUB_RELEASE -..->|Publish Release| PROD_JOBS
```

## Workflow Details

### 1. CI Workflow (`.github/workflows/ci.yml`)

**Triggers:**

- Push to `main` branch
- Pull request to `main` branch

**Jobs:**
| Job | Description |
|-----|-------------|
| `lint` | Runs linter, typecheck, and format validation |
| `build` | Builds all packages |
| `test` | Runs tests with coverage |
| `deploy-preview` | Deploys to staging-preview environment (PR only) |

**Deployment:** Preview URL based on PR title

### 2. Release Workflow (`.github/workflows/release.yml`)

**Trigger:** CI workflow completes successfully after push to `main`

**Process:**

1. Run `semantic-release`:
   - Analyzes commits for conventional commits
   - Generates changelog
   - Bumps version in `package.json`
   - Creates Git tag
   - Creates GitHub Release (draft)
2. Format files: `package.json`, `CHANGELOG.md`
3. Commits changes: `package.json`, `CHANGELOG.md`

**Semantic Release Plugins:**

- `@semantic-release/commit-analyzer`
- `@semantic-release/release-notes-generator`
- `@semantic-release/changelog`
- `@semantic-release/npm`
- `@semantic-release/exec` (format files)
- `@semantic-release/git`
- `@semantic-release/github`

### 3. Deploy Staging Workflow (`.github/workflows/deploy-staging.yml`)

**Triggers:**

- `workflow_run`: After Release workflow completes (only proceeds if version was bumped)
- Manual `workflow_dispatch`

**Jobs:**
| Job | Description |
|-----|-------------|
| `lint` | Runs linter, typecheck, and format validation |
| `build` | Builds all packages |
| `test` | Runs tests with coverage |
| `deploy-staging` | Deploy to staging environment |
| `deploy-prod-preview` | Deploy preview build to production-preview URL |

### 4. Deploy Production Workflow (`.github/workflows/deploy-production.yml`)

**Triggers:**

- GitHub Release published
- Manual `workflow_dispatch`

**Jobs:**
| Job | Description |
|-----|-------------|
| `lint` | Runs linter, typecheck, and format validation |
| `build` | Builds all packages |
| `test` | Runs tests with coverage |
| `deploy-prod` | Deploy to production environment |

### 5. Pull Request Title Validation (`.github/workflows/pull-request-title.yml`)

**Trigger:** PR opened, reopened, or edited

**Validation:** Enforces conventional commit format for PR titles

## Reusable Job Templates

| Workflow          | Purpose                            |
| ----------------- | ---------------------------------- |
| `jobs.lint.yml`   | Lint, typecheck, format check      |
| `jobs.build.yml`  | Build with turbo, upload artifacts |
| `jobs.test.yml`   | Run tests with coverage            |
| `jobs.deploy.yml` | Generic deployment job             |

## Concurrency Settings

| Workflow          | Concurrency Group            | Cancel In Progress |
| ----------------- | ---------------------------- | ------------------ |
| CI                | `main` or `ci-{pr_number}`   | Yes                |
| Release           | `main`                       | Yes                |
| Deploy Staging    | Per workflow run             | Yes                |
| Deploy Production | `${{ workflow }}-${{ ref }}` | Yes                |

## Environments

| Environment          | Purpose                  |
| -------------------- | ------------------------ |
| `staging-preview`    | Preview from PR          |
| `staging`            | Staging environment      |
| `production-preview` | Production preview build |
| `production`         | Production deployment    |

## Version Bump Decision Flow

```mermaid
flowchart LR
    A["Push to main"] --> B["CI passes?"]
    B -->|No| C["Fail"]
    B -->|Yes| D["Semantic Release"]
    D --> E["Commits since last release?"]
    E -->|feat: *| F["Minor bump + CHANGELOG"]
    E -->|fix: *| G["Patch bump + CHANGELOG"]
    E -->|feat: *<br/>BREAKING CHANGE: *| H["Major bump + CHANGELOG"]
    E -->|No conventional commits| I["No bump, no deployment"]
    F --> J["Deploy Staging & Prod Preview"]
    G --> J
    H --> J
    I --> K["End"]
    J --> L["Create GitHub Release"]
    L --> M["Deploy Production"]
```
