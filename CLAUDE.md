# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is `ansible-core`, the engine and CLI tools of Ansible (the `ansible`, `ansible-playbook`,
`ansible-galaxy`, `ansible-doc`, `ansible-vault`, etc. commands). It does **not** contain most
modules or plugins — those live in separate collection repositories. Only a small set of builtin
modules/plugins needed to bootstrap the engine live here, under `lib/ansible/modules` and
`lib/ansible/plugins`.

Requires Python >= 3.11 on the controller (see `pyproject.toml`).

## Environment setup

```shell
source ./hacking/env-setup      # runs ansible from this checkout without installing it
pip install -r requirements.txt # runtime deps
pip install -r test/lib/ansible_test/_data/requirements/units.txt  # example: test-specific deps live under test/lib/ansible_test/_data/requirements/
```

`hacking/env-setup` puts this checkout's `bin/`, `lib/`, and `test/lib/` on `PATH`/`PYTHONPATH`, so
`ansible`, `ansible-playbook`, `ansible-test`, etc. resolve to the in-tree code, not any installed
package.

## Testing

All testing goes through the **`ansible-test`** command (source at `test/lib/ansible_test/`, itself
part of this codebase). Don't invoke `pytest` directly for anything beyond one-off debugging —
`ansible-test` sets up the sanity/unit/integration environments (including Docker containers) that
CI relies on.

```shell
# Unit tests (pytest under the hood), mirrors test/units/ layout
ansible-test units test/units/module_utils/basic/test_basic.py
ansible-test units --python 3.13                      # whole suite for one Python version
ansible-test units --docker default                   # run inside the standard test container

# Sanity tests: license/boilerplate checks, pep8, pylint, mypy, validate-modules, changelog checks, etc.
ansible-test sanity --test pep8
ansible-test sanity --test pylint lib/ansible/playbook/task.py
ansible-test sanity --docker default                   # full sanity suite as CI runs it

# Integration tests (targets live under test/integration/targets/<name>)
ansible-test integration copy
ansible-test integration --docker default copy

# Only run tests affected by your changes (what CI does by default on PRs)
ansible-test units --changed
ansible-test sanity --changed
```

Unit tests live in `test/units/`, mirroring the `lib/ansible/` package layout (e.g.
`lib/ansible/playbook/task.py` <-> `test/units/playbook/test_task.py`). Integration test targets are
self-contained playbooks/roles under `test/integration/targets/<target_name>`. Sanity tests
(`test/sanity/code-smell/*.py`, plus pep8/pylint/mypy) enforce coding standards; per-file exceptions
are recorded in `test/sanity/ignore.txt`.

CI (Azure Pipelines, see `.azure-pipelines/`) runs `ansible-test sanity`, `ansible-test units` across
supported Python versions, and `ansible-test integration`/`i.sh` incidental & POSIX/Windows targets,
via the scripts in `.azure-pipelines/commands/`.

## Code conventions

- Every Python source file must start with `from __future__ import annotations` (enforced by the
  `boilerplate` sanity test) — files consisting only of top-level assignments (pure docs modules)
  are exempt.
- Modules under `lib/ansible/modules/` open with a copyright/GPL header comment, then
  `DOCUMENTATION`, `EXAMPLES`, and `RETURN` YAML-in-string blocks used to generate module docs and
  validated by the `validate-modules` and `ansible-doc` sanity tests.
- Code is formatted with `black` (`test/sanity/code-smell/black.py`); vendored code
  (e.g. `lib/ansible/_internal/_wrapt.py`) is explicitly exempted via `test/sanity/ignore.txt`.
- `pylint` and `mypy` sanity tests enforce additional static analysis; per-file/per-rule exceptions
  go in `test/sanity/ignore.txt`, not inline suppressions, unless already established in a file.
- Every user-facing change needs a changelog fragment: add a YAML file under `changelogs/fragments/`
  with one of the section keys from `changelogs/config.yaml` (`bugfixes`, `minor_changes`,
  `breaking_changes`, `deprecated_features`, `security_fixes`, etc.), e.g.:
  ```yaml
  bugfixes:
    - copy - fixed an issue where ... (https://github.com/ansible/ansible/issues/XXXXX).
  ```
- PRs targeting this repo are made against the `devel` branch; `stable-2.X` branches are
  release/maintenance branches (see `README.md`).

## Architecture

### Execution flow

1. **CLI entry points** — `bin/ansible*` scripts dispatch to `lib/ansible/cli/*.py` (`adhoc.py` for
   `ansible`, `playbook.py` for `ansible-playbook`, `galaxy.py`, `doc.py`, `vault.py`, `config.py`,
   `console.py`, `inventory.py`, `pull.py`). Each parses CLI args and builds the objects below.
2. **Inventory** (`lib/ansible/inventory/`) resolves hosts/groups/variables via inventory plugins
   (`lib/ansible/plugins/inventory/`).
3. **Playbook parsing** (`lib/ansible/playbook/`) turns YAML into `Play`, `Role`, `Block`, `Task`,
   and `Handler` objects. `playbook/base.py` and `attribute.py` implement the shared
   FieldAttribute/inheritance system all playbook objects use (vars, tags, conditionals, loops,
   etc., mixed in via `taggable.py`, `conditional.py`, `collectionsearch.py`).
4. **Execution** — `executor/playbook_executor.py` drives each `Play` through a
   `TaskQueueManager` (`executor/task_queue_manager.py`), which delegates host/task iteration order
   to a **strategy plugin** (`plugins/strategy/`: `linear`, `free`, `host_pinned`,
   `debug`). `executor/play_iterator.py` tracks per-host progress through blocks/handlers.
5. **Per-task execution** — `executor/task_executor.py` resolves the task's **action plugin**
   (`plugins/action/`, one per module family for actions needing controller-side logic) and
   **connection plugin** (`plugins/connection/`: `local`, `ssh`, `paramiko`, `winrm`, `psrp`, ...),
   applying **become** (`plugins/become/`) for privilege escalation.
6. **Module execution** — for modules that run on the target, `executor/module_common.py` builds an
   "AnsiballZ" payload: it bundles the module source with the `module_utils` it imports into a
   self-contained zipped Python script, copies it to the target via the connection plugin, and
   executes it there. `module_utils/basic.py` (`AnsibleModule`) is the runtime base class every
   builtin module uses for arg validation, fact gathering hooks, and result/exit handling.
7. **Templating** — Jinja2 expressions (`{{ }}`) are resolved through `lib/ansible/template/` and
   `_internal/_templating/`, with custom filters/tests/lookups from `plugins/filter/`,
   `plugins/test/`, `plugins/lookup/`.
8. **Results** flow back up as `TaskResult` (`executor/task_result.py`) objects and are rendered via
   **callback plugins** (`plugins/callback/`, e.g. the default stdout callback).

### Plugin system

Nearly every extension point in Ansible is a plugin type under `lib/ansible/plugins/<type>/`, loaded
by `lib/ansible/plugins/loader.py` (`PluginLoader`), which resolves plugins by name from: this
package's builtins, `ANSIBLE_LIBRARY`/config-defined paths, roles, and installed collections. When
tracing "how does X get selected/run", start at `loader.py` and the relevant `plugins/<type>/`
directory.

### Collections support

`lib/ansible/collections/` and the collection-loading machinery let content be namespaced as
`namespace.collection.plugin_name`; `_internal/ansible_collections/` holds a handful of collections
vendored in-tree (e.g. `ansible.builtin` redirects, testing shims) needed for ansible-core to
function standalone.

### `_internal` vs public modules

`lib/ansible/_internal/` holds implementation details not part of Ansible's public API (data
tagging/`_datatag`, internal templating engine bits, internal error types, ssh helpers) —
consumed by the rest of `lib/ansible/` but not meant to be imported by modules or collections.

### `ansible-test`

`test/lib/ansible_test/` is a full Python package implementing the `ansible-test` CLI itself
(command dispatch in `_internal/commands/{sanity,units,integration,coverage,shell,env}`, container
and remote provisioning in `_internal/`, support data/scripts in `_data`/`_util`). Treat it as part
of the codebase, not just tooling config, when changing how tests are run or discovered.

### Config

`lib/ansible/config/base.yml` is the canonical schema for all core configuration settings
(consumed by `config/manager.py` and surfaced via `ansible-config`); `constants.py` exposes parsed
config values used throughout the codebase.
