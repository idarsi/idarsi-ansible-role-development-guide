IDARSI-ANSIBLE-ROLE-DEVELOPMENT-GUIDE
====================================
**COPYRIGHT** 2026 ^(ida|arsi)$ collective  
**LICENSE** Creative Commons Attribution 4.0 International License [LICENSE](LICENSE)  
**AUTHORS**  
- Arsi Atomi <arsi@atomi.sh>

# IDARSI Ansible Role Development Guide

This guide defines conventions for developing reusable, maintainable, and
testable Ansible roles. It is intentionally independent of a particular
application, inventory model, operating system, or role collection.

The requirements below use these terms:

- **Required** means the role must follow the convention unless an explicit
  exception is documented.
- **Recommended** means the convention should be followed when it fits the
  role's responsibility.
- **Exception** means a deviation is acceptable only when the role documents
  why the common pattern does not apply.

## Role Contract

### Role Layout

**Required:**

- Keep the standard Ansible role structure: `defaults/`, `tasks/`,
  `templates/`, and `docs/` when examples are needed.
- Put role defaults in `defaults/main.yml`. Defaults must be safe and
  inventory-overridable.
- Use `tasks/main.yml` as the role dispatcher. It validates execution
  context, derives top-level variables, and routes by the role's state model.

**Recommended:**

- Keep file names aligned with state names, such as `started.yml`,
  `site_enabled.yml`, or `role_present.yml`.
- Keep platform-specific behavior in focused task files instead of spreading
  operating-system conditionals through every state task.

### Variable Model

**Required:**

- Use a stable role or service prefix for role variables, and reserve shared
  prefixes for genuinely shared interfaces.
- Keep public inventory keys stable once they are in use.
- Put supported operating systems and their versions in defaults so the
  supported platform contract is visible and overridable where appropriate.
- Keep proxy-related defaults explicit with `http_proxy`, `https_proxy`, and
  `no_proxy` when the role performs network operations.

**Recommended:**

- Derive task-scoped variables with `ansible.builtin.set_fact` near the top
  of the relevant task file before they are consumed.
- Keep package repository and package-manager controls explicit when a role
  manages an external repository, including certificate, GPG-check, cache,
  retry, and platform-specific interpreter settings where applicable.
- Use a structured, role-scoped input object such as
  `iac_blueprint.<service_key>` when the surrounding project uses a blueprint
  model. Do not require that naming scheme in otherwise standalone roles.

## Blueprint and Input Validation

Roles that consume structured inventory or blueprint data must validate that
input before running state-specific work.

### Validation Contract

**Required:**

- Validate the execution context and required input keys before state work
  begins. Fail fast when a required input is missing or the host platform is
  unsupported.
- Keep input validation separate from host-state validation. Input validation
  covers structure, required fields, types, allowed values, uniqueness,
  cross-record consistency, and impossible combinations.
- Run one inventory validation entry point from `tasks/main.yml` before any
  mutating state include. Validate the requested role state as well as the
  input data.
- Provide a validation-only state when the role has a structured input
  contract. It must perform no package, service, file, repository, or data
  changes and must work on a host without the role installed.
- Use `ansible.builtin.assert` with concise, actionable `fail_msg` values.
  Do not rely on an undefined-variable error as validation.

**Recommended:**

- Split non-trivial validation into focused files under `tasks/validate/`.
  Use names such as `validate_inventory.yml`, `validate_instance.yml`, or
  `validate_database.yml`.
- Use explicit `loop_control.loop_var` values for nested validators.
- Normalize optional values deliberately before asserting relationships.
- Accumulate role-prefixed validation context when a later cross-record check
  needs to compare values collected from multiple records.
- Validate the exact variables needed by each task file instead of relying on
  upstream includes to set them implicitly.

### Validation Test Contract

**Required:**

- Add validation whenever a public structured-input field is introduced or
  changed.
- Include at least one accepted input and invalid cases covering the role's
  important constraints.
- Invalid cases must assert the expected failure message, not merely that the
  play failed.
- Prefer the validation-only state so validation tests do not require package
  installation or service setup.

**Recommended:**

- Keep validation scenarios independent from scenarios that verify host
  mutation.
- Update inventory examples, the role README, supported-state documentation,
  and contribution instructions when the public input shape or state surface
  changes.

## Task Architecture

### Dispatcher and State Model

**Required:**

- Group `tasks/main.yml` into clear sections: requirement checks, variable
  setup, validation, and state dispatching.
- Keep state dispatch explicit. Prefer one `ansible.builtin.include_tasks`
  entry per supported state instead of hiding routing inside templates or
  deep conditionals.
- Use readable state names that describe the desired end state, such as
  `present`, `absent`, `started`, `instances_present`, or `site_enabled`.
- Keep substate names resource-specific and symmetric (`*_present` and
  `*_absent`) when both directions are supported.

**Recommended:**

- Split state handling into small task files such as `install.yml`,
  `present.yml`, `absent.yml`, `started.yml`, `stopped.yml`, or
  domain-specific files such as `site_present.yml` and `database_present.yml`.
- For roles managing multiple resource types, map each resource/state pair
  to an explicit task file.
- Design substates so operators can run one targeted operation remotely
  without reapplying unrelated resource sets.
- Use loop helpers such as `loop_sites.yml`, `loop_instances.yml`, or
  `loop_containers.yml` when iterating over input items.

### Includes and Shared Tasks

**Required:**

- Prefer `ansible.builtin.include_tasks` over large monolithic task files.
- Set an explicit `loop_control.loop_var` when looping over included tasks;
  nested includes must not rely on the default `item`.
- Keep each include target small and responsibility-specific.

**Recommended:**

- Use a state variable when a loop helper dispatches to multiple subtask
  files.
- Put reusable cross-role behavior in a shared task library when an existing
  implementation is available.
- Keep a role-local wrapper, such as `fs_present.yml`, `fs_absent.yml`, or
  `log_write.yml`, so the role's task surface remains explicit and can
  provide role-specific assertions or variables.

## Modules, Packages, and Commands

**Required:**

- Prefer `ansible.builtin.*` modules when they clearly express the operation.
- Prefer `command` over `shell` unless shell features are actually required.
- When using `command` or `shell`, make the reason clear from the task name
  and surrounding structure.
- Define `changed_when` and `failed_when` when command semantics are not
  safely inferred by Ansible.
- Use `ansible.builtin.template` for generated configuration.

**Recommended:**

- Keep handlers in `handlers/main.yml` when configuration changes must notify
  a service reload or restart.
- Do not hide service lifecycle changes in unrelated resource task files.

**Exception:**

- `ansible.builtin.command` or `ansible.builtin.shell` is acceptable when the
  operational interface is a CLI by design, such as `firewall-cmd`, `podman`,
  `psql`, or task-report logging.

## State Semantics and Idempotence

**Required:**

- Write tasks so rerunning the same state converges cleanly.
- Express behavior in terms of desired state, not imperative sequences.
- Keep service lifecycle states separate from resource lifecycle states.
- Keep destructive cleanup states explicit and document their scope. An
  `uninstall` state may remove packages, generated configuration, and
  managed data directories only when that behavior is intentional and
  clearly described in the role documentation.

**Recommended:**

- Separate install/uninstall from present/absent when the role distinguishes
  package bootstrap from ongoing managed configuration.
- For collection-style resources, use loop helpers that apply one item at a
  time and keep per-item task files idempotent.

## Logging and Observability

**Recommended:**

- Provide a consistent task-report or event-logging interface when the role
  is part of a larger automation system.
- Log meaningful state transitions and material side effects, such as
  package installation, service start/stop, generated files, created
  databases, granted roles, or enabled sites.
- Keep reusable logging in a dedicated `log_write.yml` task file instead of
  duplicating report-writing snippets in every state file.
- Clear temporary log variables after writing report entries so they do not
  leak into later tasks.
- Keep log messages compact and operationally useful.

## Naming and Readability

**Required:**

- Use descriptive task names that read as operator-facing execution steps.
- Keep comments rare and structural. Prefer readable task names and file
  decomposition over explanatory comment blocks inside task logic.

**Recommended:**

- Keep file names aligned with state names where possible.
- Follow the repository's existing style for section banners and file headers
  when editing an established role.

## Documentation and Testing

**Required:**

- Keep each role README aligned with the actual supported states and input
  structure.
- Add or update role examples when a role introduces a new input shape or
  state surface.
- Keep testing documentation aligned with Molecule scenarios and state
  coverage.

**Recommended:**

- Document role maturity and known coverage gaps when a role is experimental
  or not yet suitable for unattended production use.
- Document a new convention in this guide when it is intended for reuse
  across roles.
