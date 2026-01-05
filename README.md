# Build Number Action

Generate **monotonic build numbers** in GitHub Actions with **separate counters per project** (or any key).  
Implementation note: this action is a thin wrapper around the Abacus counting API.

Abacus API docs: https://abacus.jasoncameron.dev/

## Quick start

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Next build number
        id: buildno
        uses: Appegy/Abacus@v1
        with:
          # defaults to hit, shown explicitly for clarity
          operation: hit
          namespace: ci-mycompany
          key: build-api

      - run: echo "BUILD_NUMBER=${{ steps.buildno.outputs.value }}"
```

---

## Operations

Supported operations (`operation` input):

- `hit` — increment by 1 and return the new value (creates the counter if missing)
- `create` — create a counter (optionally starting at `initializer`) and return `admin_key`
- `get` — read current value (no increment)
- `info` — read value plus metadata (exists/TTL/etc.)
- `set` — set the counter to an exact value (admin key required)
- `update` — add a delta (negative allowed; admin key required)
- `reset` — reset the counter to 0 (admin key required)
- `delete` — delete the counter (admin key required)

---

## Inputs

| Name | Type | Default | Used by operations | Description |
|---|---|---|---|---|
| `operation` | string | `hit` | all | One of: `hit`, `create`, `get`, `info`, `set`, `update`, `reset`, `delete`. |
| `namespace` | string | — | all | Counter namespace (use a stable, unique prefix for your org/team). |
| `key` | string | — | all | Counter key (e.g. `build-api`, `build-web`, `build-worker`). |
| `initializer` | integer | `0` | `create` | Starting value for the counter. |
| `value` | integer | — | `set`, `update` | `set`: sets counter to `value`. `update`: adds `value` (can be negative). |
| `admin_key` | string | — | `set`, `update`, `reset`, `delete` | Admin key for management operations. Treat as a secret. |

---

## Outputs

| Output | Set by operations | Type | Notes |
|---|---|---|---|
| `value` | `hit`, `create`, `get`, `set`, `reset`, `update` | integer (string in Actions) | The counter value returned by Abacus. |
| `namespace` | `create` | string | Echoed namespace. |
| `key` | `create` | string | Echoed key. |
| `admin_key` | `create` | string | Only returned by Abacus on create. Store it in GitHub Secrets. |
| `exists` | `info` | string | Boolean-ish (`true`/`false`). |
| `expires_in` | `info` | string | Seconds-ish value from Abacus. |
| `expires_str` | `info` | string | Human-readable TTL string. |
| `full_key` | `info` | string | Fully-qualified key representation. |
| `is_genuine` | `info` | string | Boolean-ish (`true`/`false`). |
| `status` | `delete` | string | Usually `ok` on success. |
| `message` | `delete` | string | Informational message from Abacus. |

---

## Usage

### 1) Bootstrap the `admin_key` (one-time)

`admin_key` is returned **only once** by `create`. Save it immediately.  
If you lose it, **admin operations** (`set`, `update`, `reset`, `delete`) for that counter become permanently unavailable.

The workflow below:
- creates a counter
- writes `namespace`, `key`, and `admin_key` to a file
- uploads the file as an artifact so you can download it from the run page

After you download it, store the `admin_key` in **GitHub Secrets** and delete/expire the artifact (artifacts are not a secure secret store).

```yaml
name: abacus-bootstrap-admin-key

on:
  workflow_dispatch:
    inputs:
      namespace:
        description: "Abacus namespace"
        required: true
      key:
        description: "Counter key"
        required: true
      initializer:
        description: "Starting value"
        required: false
        default: "0"

jobs:
  bootstrap:
    runs-on: ubuntu-latest
    steps:
      - name: Create counter (returns admin_key once)
        id: create
        uses: Appegy/Abacus@v1
        with:
          operation: create
          namespace: ${{ inputs.namespace }}
          key: ${{ inputs.key }}
          initializer: ${{ inputs.initializer }}

      - name: Write admin key to file
        shell: bash
        run: |
          set -euo pipefail
          mkdir -p abacus
          cat > abacus/abacus-admin-key.txt << 'EOF'
          namespace=${{ inputs.namespace }}
          key=${{ inputs.key }}
          admin_key=${{ steps.create.outputs.admin_key }}
          EOF

      - name: Upload artifact (download from the run page)
        uses: actions/upload-artifact@v4
        with:
          name: abacus-admin-key-${{ inputs.namespace }}-${{ inputs.key }}
          path: abacus/abacus-admin-key.txt
          if-no-files-found: error
          retention-days: 1
```

### 2) Use all operations (except `create`) in one workflow

This workflow demonstrates:
- non-admin ops: `hit`, `get`, `info`
- admin ops: `set`, `update`, `reset`, `delete`

Prerequisite:
- add a repository secret named `ABACUS_ADMIN_KEY` that matches this `(namespace, key)` counter.

```yaml
name: abacus-operations-demo

on:
  workflow_dispatch:
    inputs:
      namespace:
        description: "Abacus namespace"
        required: true
      key:
        description: "Counter key"
        required: true
      do_delete:
        description: "Also delete the counter at the end"
        required: false
        default: "false"

jobs:
  demo:
    runs-on: ubuntu-latest
    steps:
      - name: Increment (+1)
        id: hit
        uses: Appegy/Abacus@v1
        with:
          operation: hit
          namespace: ${{ inputs.namespace }}
          key: ${{ inputs.key }}

      - name: Read current value
        id: get
        uses: Appegy/Abacus@v1
        with:
          operation: get
          namespace: ${{ inputs.namespace }}
          key: ${{ inputs.key }}

      - name: Read metadata (exists/TTL/etc.)
        id: info
        uses: Appegy/Abacus@v1
        with:
          operation: info
          namespace: ${{ inputs.namespace }}
          key: ${{ inputs.key }}

      - name: Set exact value (admin)
        id: set
        uses: Appegy/Abacus@v1
        with:
          operation: set
          namespace: ${{ inputs.namespace }}
          key: ${{ inputs.key }}
          value: 1000
          admin_key: ${{ secrets.ABACUS_ADMIN_KEY }}

      - name: Apply delta (admin)
        id: update
        uses: Appegy/Abacus@v1
        with:
          operation: update
          namespace: ${{ inputs.namespace }}
          key: ${{ inputs.key }}
          value: -10
          admin_key: ${{ secrets.ABACUS_ADMIN_KEY }}

      - name: Reset to 0 (admin)
        id: reset
        uses: Appegy/Abacus@v1
        with:
          operation: reset
          namespace: ${{ inputs.namespace }}
          key: ${{ inputs.key }}
          admin_key: ${{ secrets.ABACUS_ADMIN_KEY }}

      - name: Delete counter (admin)
        if: ${{ inputs.do_delete == 'true' }}
        id: delete
        uses: Appegy/Abacus@v1
        with:
          operation: delete
          namespace: ${{ inputs.namespace }}
          key: ${{ inputs.key }}
          admin_key: ${{ secrets.ABACUS_ADMIN_KEY }}
```

---

## Notes

- `hit` consumes a number immediately. If your job fails later, that number is still consumed.
- For multi-project CI, use different `key` values (e.g. `build-api`, `build-web`) to keep counters independent.
- If you delete a counter, you must `create` it again, and the new counter will have a **new** `admin_key`.
