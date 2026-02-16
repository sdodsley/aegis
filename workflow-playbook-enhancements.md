# Workflow and Playbook Enhancement Proposals

This document captures concrete next-step enhancements for the current DR and discovery playbooks.

## 1) Reliability and safety improvements (high priority)

1. **Enforce input contracts everywhere**
   - Add preflight validation for optional-but-critical variables used during instance boot (`os_instance_name`, `os_flavor`, `os_network`, `target_host`, `target_volume_type`) whenever `bootable: true`.
   - Add `failed_when` checks after data lookup tasks (for example, fail if `os_volume_info.volumes | length == 0`).

2. **Add deterministic snapshot/clone selection policy**
   - In async recovery, snapshot selection currently depends on the order returned by the API.
   - Improve safety by selecting the newest snapshot by `created` timestamp (or explicitly filtering by a provided `snapshot_name`/`snapshot_prefix`).

3. **Avoid naming collisions during repeated runs**
   - Use a unique suffix (timestamp or run id) for restored/managed volume names.
   - Example: `{{ target_os_volume_name }}-{{ ansible_date_time.iso8601_basic_short }}`.

4. **Protect against accidental production targeting**
   - Add a confirmation guard variable such as `dr_confirm: false` and require it to be set `true` before mutation tasks run.

## 2) Operational observability (high priority)

1. **Structured recovery report artifact**
   - Persist a JSON/YAML report summarizing source volume metadata, snapshot used, managed Cinder volume id, and booted server id.
   - Save into a local `artifacts/` directory for auditing and handoff.

2. **Verbose run phases and timing**
   - Wrap major sections in `block:` with clear phase names and include elapsed-time debug output.

3. **Post-recovery smoke checks**
   - Verify resulting volume status (`available`/`in-use`) and server status (`ACTIVE`) with retries.
   - Optionally test TCP reachability of the recovered server's IP.

## 3) Security and secret handling (medium priority)

1. **Move secrets out of playbook vars**
   - Read `fa_api_token` from environment variables or Ansible Vault.
   - Document usage examples for `--extra-vars` and vault-encrypted var files.

2. **Mask sensitive data in logs**
   - Apply `no_log: true` to tasks that include API tokens or auth objects.

## 4) Reusability and maintainability (medium priority)

1. **Convert duplicated logic into a role**
   - Create a `roles/openstack_dr_recovery` role shared by sync/async playbooks.
   - Keep transport-specific steps as `include_tasks` files (`async.yml`, `sync.yml`).

2. **Standardize variable names and schema**
   - Keep naming consistent across playbooks (`tenant_name`, `target_volume_type`, `target_host`).
   - Provide a single `defaults/main.yml` and `README` input table.

3. **Tag tasks by lifecycle stage**
   - Use tags like `precheck`, `snapshot`, `manage`, `boot`, `verify` to allow partial execution.

## 5) Topology capture and rebuild readiness (for `nova-instance-details.yaml`)

1. **Export machine-readable inventory**
   - In addition to debug output, write server/security-group/network details into structured files.

2. **Capture additional dependencies**
   - Gather router, subnet, floating IP, port, and keypair details to enable fuller environment reconstruction.

3. **Add selective scope controls**
   - Support filters for projects, availability zones, name patterns, and status.

## 6) CI quality gates (recommended)

1. **Add linting and syntax checks**
   - Run `ansible-playbook --syntax-check` and `ansible-lint` in CI.

2. **Add Molecule or mocked integration tests**
   - Build minimal offline checks for variable validation and task flow.

3. **Add pre-commit hooks**
   - Include YAML formatting and lint rules to prevent drift.

## Suggested phased rollout

- **Phase 1 (quick wins):** stronger preflight asserts, variable consistency fixes, lint/syntax checks.
- **Phase 2:** structured report outputs, retry-based post checks, task tagging.
- **Phase 3:** role refactor and richer topology capture for reproducible DR rehearsals.
