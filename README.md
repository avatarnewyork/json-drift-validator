# JSON Drift Validator

**JSON Drift Validator** is a GitHub Action that detects configuration drift between **two JSON datasets**, even when deeply nested.

The action flattens JSON into dot-path key/value pairs (e.g., `dependencies.react`, `settings.cache.mobile`), applies wildcard-based severity rules, and flags configuration drift as:

- **Critical** — fail the workflow
- **Medium** — warn but continue
- **Ignored** — intentionally skipped, takes highest precedence

This action is ideal for:

- WordPress configuration exports (e.g., **WP Rocket**, **ACF**, **WooCommerce**)
- Terraform JSON plan/state validation
- CloudFormation, Kubernetes, or SaaS JSON config comparison
- Any config-as-code environment where drift detection is needed

It now supports three ways to provide JSON input:

1. **JSON files** – recommended  
2. **Base64-encoded JSON strings** – recommended when using secrets  
3. **Raw JSON strings** – discouraged due to escaping issues

---

## 🔥 Key Features

- **Flattened JSON diffing** using stable dot-path keys  
- **Wildcard pattern matching** for severity rules  
  - Examples: `cache.*`, `database.settings.*`, `resource_changes[*].change.after.*`
- **Ignore list overrides** critical and medium rules  
- **Flexible input formats**: File, Base64, Raw JSON
- **Works with extremely large JSON files**
- **Generates a Markdown drift report** and uploads as an artifact  
- **General-purpose drift detection** — not tied to any specific tool  

---

## 📥 Inputs

### JSON Input Options

You MUST provide **one** baseline JSON input and **one** comparison JSON input.  
Use either:

### **1. File-based JSON (recommended)**

| Input | Description |
|-------|-------------|
| `baseline_json_file` | Path to a file containing baseline JSON |
| `compare_json_file` | Path to a file containing comparison JSON |

---

### **2. Base64 JSON**

| Input | Description |
|-------|-------------|
| `baseline_json_base64` | Base64-encoded baseline JSON |
| `compare_json_base64` | Base64-encoded comparison JSON |

Useful when reading JSON from secrets, artifacts, or encrypted exports.

---

### **3. Raw JSON (discouraged)**

| Input | Description |
|-------|-------------|
| `baseline_json` | Raw JSON string |
| `compare_json` | Raw JSON string |

Raw JSON is fragile due to quoting, newlines, `%` handling, and GitHub’s workflow syntax. Prefer **file** or **base64**.

---

## 🎯 Severity Rule Inputs

Each input accepts **comma-separated wildcard patterns**.

| Input | Purpose |
|--------|---------|
| `critical_keys` | Any matched key drift **fails the workflow** |
| `medium_keys` | Matched drift **warns** |
| `ignore_keys` | Matched keys are **ignored**, even if also in critical/medium |

### Wildcard Examples

| Pattern | Matches |
|---------|---------|
| `cache.*` | `cache.mobile`, `cache.ssl`, `cache.settings.mode` |
| `database.*` | All keys under `database` |
| `resource_changes[*].change.after.*` | Any Terraform resource attribute |
| `**.version` | Any nested `version` key |
| `metadata.*` | All metadata fields |

---

## 📤 Outputs

| Output | Meaning |
|--------|---------|
| `critical_fail` | `"1"` if any critical rule was violated |
| `medium_warn` | `"1"` if any medium rule was violated |

A Markdown report is uploaded as artifact:  
**`json-drift-report/drift-report.md`**

---

# 🚀 Usage Examples

Below are several complete examples reflecting real-world usage.

---

# ✅ **Example: Simplest Possible (JSON file inputs)**

```yaml
jobs:
  drift:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - run: echo '{"a":1,"b":2}' > baseline.json
      - run: echo '{"a":2,"b":2}' > compare.json

      - name: Validate drift
        id: drift
        uses: avatarnewyork/json-drift-validator@v2
        with:
          baseline_json_file: baseline.json
          compare_json_file: compare.json
          critical_keys: "a"
```

---

# 🟦 **Example: Base64 inputs (recommended for secrets)**

```yaml
- name: Validate drift
  id: drift
  uses: avatarnewyork/json-drift-validator@v2
  with:
    baseline_json_base64: ${{ secrets.MY_BASELINE_B64 }}
    compare_json_base64: ${{ steps.build.outputs.json_base64 }}

    critical_keys: "config.*"
    medium_keys: "metadata.version"
    ignore_keys: "timestamps.*,logs.*"
```

---

# 🟥 **Example: Raw JSON (discouraged but supported)**

```yaml
- uses: avatarnewyork/json-drift-validator@v2
  with:
    baseline_json: '{"cache": {"ssl": true}}'
    compare_json: '{"cache": {"ssl": false}}'
    critical_keys: "cache.ssl"
```

---

# 🧩 **Example: WP Rocket Configuration Drift Check**

```yaml
jobs:
  wprocket:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Decode WP Rocket export to file
        run: |
          echo "${{ steps.export.outputs.wp_export_base64 }}" | base64 -d > wp_export.json

      - name: Validate WP Rocket Settings
        id: validate
        uses: avatarnewyork/json-drift-validator@v2
        with:
          baseline_json_base64: ${{ secrets.WP_ROCKET_BASELINE_B64 }}
          compare_json_file: wp_export.json

          critical_keys: >
            cache_*,
            minify_*,
            delay_js_*,
            lazyload.*,
            heartbeat.*,
            cdn.*,
            database.*

          medium_keys: "version,previous_version"
          ignore_keys: "secret_*,consumer_*,license"
```

---

# 🌐 **Example: Terraform Drift Detection**

```yaml
with:
  baseline_json_file: terraform-baseline.json
  compare_json_file: terraform-output.json

  critical_keys: "resource_changes[*].change.after.*"
  medium_keys: "terraform_version,format_version"
  ignore_keys: "resource_changes[*].address"
```

---

## 🔍 How Drift Detection Works

1. JSON is flattened into `key<TAB>value` lines:  
   ```
   dependencies.react   18.2.0
   cache.mobile.enable  true
   ```
2. Keys are matched against your wildcard rules:
   - If key matches `ignore_keys` → **ignored**
   - Else if matches `critical_keys` → **fail**
   - Else if matches `medium_keys` → **warn**
   - Else → ignored
3. Differences appear in `drift-report.md`

Example drift output:

```
❌ Critical: cache.mobile
- Baseline: true
- Compare:  false

⚠️ Medium: version
- Baseline: 1.0.3
- Compare:  1.0.4
```

---

# 🧪 Local Testing

To test flattening:

```sh
jq -r '
  paths(scalars) as $p |
  [
    ($p | map(if type=="number" then "[\(.|tostring)]" else tostring end) | join(".")),
    (getpath($p) | tostring)
  ] | @tsv
' file.json
```

To simulate workflow locally:

```sh
brew install act
act -j drift
```

---

# 🤝 Contributing

See `CONTRIBUTING.md`.  
Pull requests and enhancements are welcome, especially rule presets for tools (WP Rocket, Terraform, package.json, etc.).

---

# 🛡 Security

Do not include secrets or production configuration in issues or pull requests.  
See `SECURITY.md` for safe disclosure guidelines.

---

# 📄 License

MIT License.  
See `LICENSE` for details.
