IDARSI-ANSIBLE-ROLE-TESTING-GUIDE
=================================
**COPYRIGHT** 2026 ^(ida|arsi)$ collective<br>
**LICENSE** Creative Commons Attribution 4.0 International License [LICENSE](LICENSE)<br>
**AUTHORS**<br>
- Arsi Atomi <arsi@atomi.sh>

# Idarsi Ansible Role Testing Guide

This guide describes how to build reliable automated tests for Ansible roles.
It complements the [Idarsi Ansible Role Development Guide](ansible-role-development-guide.md), which defines general role design and implementation conventions.

The guide is written for reusable Ansible roles. Idarsi-specific blueprint
requirements are called out where they add to the general guidance. Test
environment installation and backend setup belong to the
[`idarsi-ansible-test-environments`](https://github.com/idarsi/idarsi-ansible-test-environments)
documentation.

## Testing goals

A role's automated suite should provide evidence that its documented contract
works, including supported inputs, states, platforms, and important safety
properties. A passing happy-path test alone is not evidence of production
readiness.

Keep the complete automated suite comprehensive while keeping per-change CI
fast. The full suite should be runnable from the role repository using its
documented test environment. CI may run a focused pull-request set and reserve
expensive scenarios or broad matrices for scheduled or release runs.

## Define the support contract

Read the role's README and implementation before designing the matrix. For
Idarsi IaC roles, compare the README with `iac_supported_os` and platform
validation. Keep those sources consistent.

Distinguish clearly between:

- **Supported platforms:** operating system and version combinations the role
  claims to support.
- **Automated coverage:** combinations actually exercised by automated tests,
  including whether they run in CI, on a schedule, or only manually.

Every supported platform/version combination should have baseline automated
evidence. If that is impractical, narrow the support claim or explain the gap;
do not imply that a passing container on one release proves support for every
release in its family.

Roles should state their supported Ansible Core range. Test at least the
minimum supported line and the normal development line. A forward-compatibility
line can be tested as an early warning and treated as non-blocking until the
role adopts it. Idarsi's shared environment names these lanes `ansible-min`,
`ansible-current`, and `ansible-next`; those environment labels do not by
themselves define a role's support promise.

## Build the role test suite

Keep Molecule scenarios focused on behavior that differs materially. Common
scenario purposes include:

```text
validation
baseline
lifecycle
guardrails
all_absent
platform-matrix
cluster
replication
upgrade
```

Do not create every scenario for every role. Avoid an oversized scenario that
combines unrelated behavior, and avoid a Cartesian product of every platform,
application version, and optional feature. Choose combinations that exercise
meaningful compatibility boundaries and risk.

### Validation

Structured-input roles need tests for accepted and rejected inputs. Cover
important type, required-field, value, uniqueness, reference, and
cross-field constraints. Assert the expected failure message for invalid
input.

For Idarsi IaC roles, follow the development guide's validation contract and
prefer the `validate` state. Validation tests should not need package
installation, service startup, or application data changes.

### Baseline and idempotence

Test the smallest supported input and the normal stable use path. Verify
observable results such as installed packages, configuration, ownership,
permissions, services, and resources. Use a baseline scenario for every
supported platform/version combination.

Stable convergence must be idempotent. Run convergence twice and assert the
second run has no unexpected changes. Do not weaken idempotence checks to hide
role defects; isolate and explain unavoidable change reporting from external
tools.

### Public states and features

Test the public states and important features that materially change role
behavior. Add lifecycle tests for targeted states, and verify that a targeted
operation does not change unrelated resources. Test important application,
dependency, and optional-feature version boundaries without testing every
theoretical combination.

For multi-host roles, verify relationships and runtime behavior between hosts
when practical. Configuration-file checks alone are insufficient when
connectivity, leader election, replication, or failover can be observed.

### Destructive behavior and guardrails

Test states that remove data, files, services, packages, accounts, or other
persistent resources in disposable environments. Cover both safe cleanup and
rejection of unsafe inputs. Assert the expected error before destructive work
begins.

Keep test-only paths, credentials, containers, service mocks, and other
overrides in Molecule fixtures. Use fake secrets and do not copy test-only
settings into production examples.

### Security and check mode

Test security-sensitive defaults and dangerous opt-ins when they are part of
the role contract. Use fake secrets and suppress diffs for secret-bearing
content.

Test check and diff mode for representative stable states when the managed
operations support it. Do not claim preview support for tools that cannot
safely preview changes; document the limitation instead.

Use `verify.yml` for machine-verifiable observable results. Avoid assertions
that only repeat the input variables supplied to the role.

## Platform and Ansible version coverage

Use a matrix that gives meaningful evidence without multiplying every test by
every dimension. Include the role's supported platforms, relevant application
versions, and supported Ansible Core boundaries. The `TESTING.md` matrix must
show what actually runs for each meaningful combination.

For Idarsi roles, use the shared environment's pinned tool versions for local
Molecule and Ansible Lint runs. The shared Ansible Core lines are useful test
lanes, but each role remains responsible for documenting its own compatibility
range.

## CI lanes

Keep required pull-request checks quick and useful. A typical pull-request set
contains:

- Ansible syntax checking;
- production-profile Ansible Lint;
- validation and a baseline scenario on the normal development Ansible line;
- the scenario for the changed feature or state, when applicable.

Run the comprehensive automated Molecule suite on a schedule or before a
release. Include the full set of documented automated scenarios and broader
platform/application matrices. Also run a focused compatibility set on the
minimum supported Ansible line and, as a forward-compatibility probe, on the
next line. Keep the forward-compatibility probe non-blocking until that line is
adopted as supported.

Run scenarios requiring a full VM, SELinux, firewall behavior, or realistic
host networking on KVM or another suitable backend. If KVM is not available in
CI, label that coverage manual and document its frequency, prerequisites, and
limitations. Do not describe manual coverage as an automated CI check.

## Test documentation

Keep `TESTING.md` aligned with the scenarios and workflows that actually run.
Include an automated test matrix with these columns:

```text
Platform/image | Ansible or application versions | Molecule scenarios | Main coverage
```

Describe every scenario's observable coverage and label it as pull-request,
scheduled, release, or manual. Document commands, prerequisites, limitations,
and known coverage gaps. State explicitly when a scenario covers multiple
images or versions. Do not claim coverage that is not executed.

## Syntax and lint

Every role should pass syntax checking and production-profile Ansible Lint.
For example:

```bash
ansible-playbook --syntax-check \
  -i docs/inventory-minimal.yml \
  docs/playbook-example.yml
ansible-lint --profile production
```

Do not disable lint rules globally to hide avoidable problems. Document
justified exceptions.

## Local Idarsi test environments

Idarsi role repositories keep their scenarios, fixtures, and role-specific
collections in the role repository. Use the shared
`idarsi-ansible-test-environments` setup for local tools and backend lifecycle
support. Use Podman by default; use KVM when the scenario requires full virtual
machines or host behavior that containers cannot represent. Follow that
repository's setup and scenario integration instructions. Do not use Docker
for Molecule tests.

## Test development workflow

For a new role:

1. Read the role contract, supported platforms, and input schema.
2. Add validation coverage for structured inputs.
3. Test minimal supported input and baseline behavior on every supported
   platform/version combination.
4. Verify idempotence.
5. Add focused tests for important public states, features, and compatibility
   boundaries.
6. Add guardrail and destructive-operation tests where applicable.
7. Add multi-host, security, and check-mode coverage where they are part of the
   role contract.
8. Add syntax, lint, and CI lanes.
9. Document the complete automated suite, CI cadence, and any gaps in
   `TESTING.md`.

When improving an existing role, compare its support claims with the tests,
documentation, and CI. Add the smallest set of assertions or scenarios that
closes meaningful gaps, and avoid duplicating equivalent coverage.
