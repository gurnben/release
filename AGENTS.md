# AGENTS.md

Guidance for AI agents working in the `openshift/release` repository. This repository holds OpenShift CI workflow configuration, cluster manifests, and component build definitions for OKD and OCP.

## Essential Commands

### Day-to-Day Development

```bash
make update                    # Regenerate ALL downstream artifacts (run after any config change)
make jobs                      # Regenerate Prow job configs from ci-operator configs
make ci-operator-config        # Normalize/determinize ci-operator config files
make prow-config               # Normalize Prow configuration
make registry-metadata         # Regenerate step-registry metadata
make release-controllers       # Regenerate release controller configurations
make boskos-config             # Regenerate Boskos resource configs
make new-repo                  # Interactive: set up CI for a new repository (then auto-runs make update)
```

### Validation

```bash
make check                     # Run ALL validation checks
make check-core                # Validate core-services/ configs
make check-services            # Validate services/ configs
make checkconfig               # Validate Prow configuration (runs in container)
make ci-operator-checkconfig   # Validate ci-operator configs against step-registry
make validate-step-registry    # Validate step-registry definitions
make python-validation         # Run pylint on hack/ scripts
```

### Container Engine

Default is `podman`. Override with:

```bash
export CONTAINER_ENGINE=docker
```

Most `make` targets run containerized tools from `quay.io/openshift/ci-public`. Set `SKIP_PULL=true` to avoid re-pulling images.

### Python Environment

Some targets (e.g., `release-controllers`, `boskos-config`) require Python with `pyyaml`:

```bash
python3 -m venv venv/
source venv/bin/activate
python3 -m pip install pyyaml
```

## Repository Structure

```
ci-operator/
├── config/               # Source of truth: CI configs per org/repo/branch (manually edited)
│   └── <org>/<repo>/     # e.g., openshift/installer/
│       ├── <org>-<repo>-<branch>.yaml
│       └── <org>-<repo>-<branch>__periodics.yaml   # Variant periodic configs
├── jobs/                 # AUTO-GENERATED Prow job YAML (never edit manually)
├── step-registry/        # Reusable CI steps, chains, and workflows (4,400+ components)
│   └── <component>/      # e.g., ipi/install/install/
│       ├── *-ref.yaml              # Step definition (metadata)
│       ├── *-commands.sh           # Step script
│       ├── *-chain.yaml            # Chain (ordered step sequence)
│       ├── *-workflow.yaml         # Workflow (pre/test/post phases)
│       ├── *.metadata.json         # Auto-generated metadata
│       └── OWNERS
├── templates/            # LEGACY - do not add new templates, use step-registry instead
└── lease/                # Lease configurations
core-services/            # Core infrastructure configs (auto-applied by Prow postsubmit)
services/                 # Additional service configs
clusters/                 # Cluster manifests (build farms, app.ci, hosted-mgmt)
projects/                 # Experimental/legacy service manifests
tools/                    # Container image build manifests for tooling
hack/                     # Scripts: validation, branch management, secret management
docs/                     # Documentation
```

## Critical Rules

1. **Never manually edit `ci-operator/jobs/`** — always edit `ci-operator/config/` and run `make update`
2. **Always run `make update` after config changes** — this generates Prow jobs, normalizes configs, and updates metadata
3. **`zz_generated_metadata`** in config files is auto-generated — do not manually edit
4. **Config files are deterministically formatted** — manual formatting will be overwritten by generation tools
5. **New test workflows must use step-registry**, not templates (templates are legacy)
6. **Core-services configs are auto-applied** by the `config-updater` Prow plugin after merge

## CI-Operator Config File Format

Config files in `ci-operator/config/<org>/<repo>/` follow the naming convention `<org>-<repo>-<branch>.yaml` and define:

```yaml
base_images:       # Base images to import
build_root:        # Image used for builds
images:            # Container images to build
releases:          # OpenShift releases to test against (initial, latest)
resources:         # CPU/memory resource requests
tests:             # Test definitions (reference step-registry workflows)
promotion:         # Where to push successful builds
zz_generated_metadata:  # AUTO-GENERATED — do not edit
```

### Variant Periodic Configuration

Periodic tests can be separated into `<org>-<repo>-<branch>__periodics.yaml` files:
- Contains only tests with `interval:` or `cron:` scheduling
- Maintains release-specific periodic configuration consumable by CI analytical tooling
- Must duplicate `base_images`, `build_root`, `images`, `promotion`, `releases` from main config
- Generated jobs go to separate `-periodics.yaml` files in `ci-operator/jobs/`

## Step Registry Components

### Three Primitives

| Type | File Pattern | Purpose |
|------|-------------|---------|
| **Ref (Step)** | `*-ref.yaml` + `*-commands.sh` | Atomic task with script |
| **Chain** | `*-chain.yaml` | Ordered sequence of refs/chains |
| **Workflow** | `*-workflow.yaml` | Complete test: `pre` → `test` → `post` phases |

### Naming Convention

Component names are dash-joined path segments. Example:
```
ci-operator/step-registry/ipi/install/install/
├── ipi-install-install-ref.yaml
├── ipi-install-install-commands.sh
└── ipi-install-install-ref.metadata.json
```

### Step Script Best Practices

- Default to `set -euo pipefail` (without `-x`)
- Temporarily disable tracing around sensitive operations (passwords, tokens, URLs)
- Use `${SHARED_DIR}` for sharing data between steps, not stdout
- Never echo credentials to logs

```bash
# Protect sensitive operations:
[[ $- == *x* ]] && WAS_TRACING=true || WAS_TRACING=false
set +x
secret=$(oc get secret --template='{{.data.password}}' | base64 -d)
echo "$secret" > "${SHARED_DIR}/password"
$WAS_TRACING && set -x
```

### Adding a New Step

1. Create files in `ci-operator/step-registry/<component>/`:
   - `<name>-ref.yaml` — step metadata (as, commands, credentials, env, from_image, resources, timeout)
   - `<name>-commands.sh` — script to execute
2. Run `make registry-metadata`
3. Reference in workflow chains or directly in configs

### Creating a New Workflow

1. Define individual steps if needed
2. Create `<name>-workflow.yaml` with `pre`, `test`, `post` phases
3. Run `make validate-step-registry`

## File Naming Conventions

| Location | Pattern | Example |
|----------|---------|---------|
| `ci-operator/config/` | `<org>-<repo>-<branch>.yaml` | `openshift-installer-release-4.21.yaml` |
| `ci-operator/config/` (periodic) | `<org>-<repo>-<branch>__periodics.yaml` | `openshift-installer-release-4.21__periodics.yaml` |
| `ci-operator/jobs/` | `<org>-<repo>-<org>-<repo>-<branch>-<type>.yaml` | Auto-generated |
| `step-registry/` | `<component-name>-{ref,chain,workflow}.yaml` | `ipi-install-install-ref.yaml` |
| `core-services/` | `admin_*.yaml` for admin resources | `admin_cluster-reader-1_list.yaml` |

## Configuration Generation Pipeline

1. `ci-operator/config/` — source of truth (manually edited)
2. `ci-operator-prowgen` — generates `ci-operator/jobs/` from config
3. `determinize-ci-operator` — normalizes config formatting
4. `determinize-prow-config` — normalizes Prow config
5. All tools run containerized from `quay.io/openshift/ci-public`

## Release Versioning

OpenShift uses semantic versioning with minor releases (4.18, 4.19, ..., 4.22):
- Branch names: `release-4.21`, `release-4.20`, or `main`/`master`
- Config files: one per branch per repository

### Config Brancher

```bash
config-brancher --config-dir ./ci-operator/config \
  --current-release 4.21 --future-release 4.22 --confirm
```

- `--current-release`: the version that `main`/`master` currently targets
- `--future-release`: creates an additional config for the next version
- `--skip-periodics`: avoids branching periodic jobs (use when periodics are in `__periodics.yaml` files)

## OWNERS Files

Standard Prow OWNERS format with `approvers:` and `reviewers:` lists. The top-level `OWNERS_ALIASES` file defines ~50+ team aliases (e.g., `dptp`, `staff-engineers`, `openstack-approvers`).

## Prow Configuration

- Main config: `core-services/prow/02_config/_config.yaml`
- Plugin config: `core-services/prow/02_config/_plugins.yaml`
- Per-org overrides: `core-services/prow/02_config/<org>/_prowconfig.yaml`

## Code Style

### YAML

- Deterministic formatting enforced by generation tools
- Indentation validated by `hack/validate-yaml-indentation.sh` (run via `make check`)

### Python

- Pylint config at `hack/.pylintrc`: max-line-length=150, snake_case naming
- Run via `make python-validation`

### Shell Scripts (step-registry)

- Use `set -euo pipefail`
- Follow the tracing disable pattern for sensitive operations
- Multi-cloud credential handling patterns visible in existing steps (e.g., `ipi-install-install-commands.sh`)

## Useful Hack Scripts

| Script | Purpose |
|--------|---------|
| `hack/validate-boskos.sh` | Validate Boskos configuration |
| `hack/validate-yaml-indentation.sh` | Check YAML indentation |
| `hack/validate-labels.py` | Validate label configuration |
| `hack/validate-cluster-profiles-config.py` | Validate cluster profiles |
| `hack/ci-secret-bootstrap.sh` | Bootstrap CI secrets |
| `hack/ci-secret-generator.sh` | Generate CI secrets |
| `hack/job.sh` | Run a Prow job locally |
| `hack/annotate.sh` | Annotate release controller definitions |
| `hack/generators/release-controllers/generate-release-controllers.py` | Generate release controller configs |

## Claude Code Integration

### Slash Commands (`.claude/commands/`)

| Command | Purpose |
|---------|---------|
| `/step-finder <query>` | Search step-registry for existing components (always use before creating new ones) |
| `/effective-env` | Resolve cascading env vars through workflow→chain→step hierarchy |
| `/rehearse-debug-cluster` | Add wait ref to test config and trigger rehearsal |
| `/migrate-variant-periodics` | Migrate periodic configs between releases |
| `/find-missing-variant-periodics` | Identify missing periodic configs for a release |
| `/payload-job-add` | Add job to release payload (always informing/optional initially) |
| `/payload-job-promote` | Promote informing job to blocking |
| `/payload-job-demote` | Demote blocking job to informing |
| `/payload-job-aggregate` | Add aggregation to a payload job |
| `/payload-job-deaggregate` | Remove aggregation from a payload job |

### Helper Scripts (`.claude/scripts/`)

| Script | Purpose |
|--------|---------|
| `effective_env.py` | Resolves env vars through workflow→chain→step hierarchy with override tracking |
| `migrate_periodic_file.py` | Transforms periodic config YAML between versions |

## Key External References

- [OpenShift CI Documentation](https://docs.ci.openshift.org/)
- [CI Operator Reference](https://steps.ci.openshift.org/ci-operator-reference)
- [Contributing Guide](https://docs.ci.openshift.org/docs/how-tos/contributing-openshift-release/)
- [Onboarding New Components](https://docs.ci.openshift.org/docs/how-tos/onboarding-a-new-component/)

## Gotchas

1. **`make update` is slow** — it pulls and runs multiple containers; use `SKIP_PULL=true` to avoid re-pulling images
2. **macOS requires `gsed` and `gtimeout`** — some Makefile targets use `sed`/`timeout` with GNU-specific flags
3. **Config files are auto-normalized** — don't worry about manual YAML formatting; tools will rewrite it
4. **`ci-operator/jobs/` diffs are large** — a small config change can regenerate many job files; this is expected
5. **Step-registry has 4,400+ components** — always search before creating new ones to avoid duplication
6. **Periodic configs need release-specific files** — use `__periodics.yaml` variant files for CI analytical tooling, not the main config
7. **Release branch configs are managed by `config-brancher`** — don't manually maintain release branch configs that are derived from `main`
8. **Templates are legacy** — never add new `ci-operator/templates/`; use step-registry workflows
9. **Some subdirectories have their own AGENTS.md** — check for `AGENTS.md` files in specific config directories (e.g., `ci-operator/config/openshift/openshift-tests-private/AGENTS.md`) for domain-specific guidance
