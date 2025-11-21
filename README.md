# JSON Drift Validator

**JSON Drift Validator** is a GitHub Action that detects configuration drift between **two JSON datasets**, even when deeply nested.

The action flattens JSON into dot-path key/value pairs (e.g., `dependencies.react`, `settings.cache.mobile`), applies wildcard-based severity rules, and flags configuration drift as:

- **Critical** — listed in the report, **counted**, and they now **fail the workflow automatically**
- **Medium** — listed in the report and counted as warnings
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
- Supports **JSON via file path**, **base64**, or **raw JSON string**
- Works with large & deeply nested JSON configs
- **Automatic workflow failure** when critical drift is detected
- **Readable drift-summary** written to console:
  ```
  JSON Drift Summary:
  Critical drift count: 2
  Medium drift count:   1
  ```
- **Markdown drift report** uploaded as an artifact  
- Fully general-purpose — works with any JSON-based config

---

## 📥 Inputs

You must provide one baseline JSON input and one comparison JSON input:

### **1. File-based JSON (recommended)**
| Input | Description |
|-------|-------------|
| `baseline_json_file` | Path to baseline JSON file |
| `compare_json_file` | Path to comparison JSON file |

---

### **2. Base64 JSON**
| Input | Description |
|-------|-------------|
| `baseline_json_base64` | Base64-encoded baseline JSON |
| `compare_json_base64` | Base64-encoded comparison JSON |

---

### **3. Raw JSON (discouraged)**

| Input | Description |
|-------|-------------|
| `baseline_json` | Raw baseline JSON string |
| `compare_json` | Raw comparison JSON string |

Raw JSON is fragile in GitHub workflows due to quotation and newline escaping. Prefer file or base64.

---

## 🎯 Severity Rule Inputs

Each list accepts **comma-separated wildcard patterns**:

| Input | Purpose |
|--------|---------|
| `critical_keys` | Drift **fails the workflow**, counts as critical |
| `medium_keys` | Drift **warns**, counts as medium |
| `ignore_keys` | Drift fully ignored; overrides all other rules |

### Wildcard Examples

| Pattern | Matches |
|---------|---------|
| `cache.*` | `cache.mobile`, `cache.ssl`, etc. |
| `database.*` | All database settings |
| `resource_changes[*].change.after.*` | Terraform resource drift |
| `**.version` | Any version key anywhere |
| `metadata.*` | Metadata fields |

---

## 📤 Outputs

| Output | Meaning |
|--------|---------|
| `critical_fail` | Number of critical drift violations |
| `medium_warn` | Number of medium drift warnings |

> **Note:**  
> Even if the workflow fails due to critical drift,  
> the drift report artifact is still uploaded.

---

# 🚀 Usage Examples

Below are several complete examples reflecting real-world usage.

---

# ✅ Example: Basic File-Based Drift Check

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

This will fail the workflow because key `a` changed.

---

# 🟦 Example: Base64 Inputs (recommended for secrets)

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

# 🟥 Example: Raw JSON (discouraged but supported)

```yaml
- uses: avatarnewyork/json-drift-validator@v2
  with:
    baseline_json: '{"cache": {"ssl": true}}'
    compare_json: '{"cache": {"ssl": false}}'
    critical_keys: "cache.ssl"
```

---

# 🧩 WP Rocket Configuration Drift Check

This example uses **base64 input**, **automatic failure** on critical drift, and the updated validator.

```yaml
- name: Validate WP Rocket Settings
  id: validate
  uses: avatarnewyork/json-drift-validator@v2
  with:
    baseline_json_base64: ${{ secrets.WP_ROCKET_BASELINE_BASE64 }}
    compare_json_base64: ${{ env.WP_EXPORT_BASE64 }}

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

Output example:

```
JSON Drift Summary:
Critical drift count: 2
Medium drift count:   1
❌ Critical drift detected. Failing the action.
```

The workflow fails, and `drift-report.md` is uploaded.

---

# 🌐 Terraform Example

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

1. JSON is flattened to key/value TSV lines:
   ```
   cache.mobile         true
   cache.ssl            false
   version              2
   ```
2. Keys match severity rules:
   - `ignore_keys` → excluded  
   - `critical_keys` → counted + fail  
   - `medium_keys` → counted + warn  
3. The report lists differences with severity, e.g.:

```
❌ Critical: cache.mobile
- Baseline: true
- Compare:  false

⚠️ Medium: version
- Baseline: 1.0.3
- Compare:  1.0.4
```

4. Summary appended automatically:
```
### Summary
- Critical drift items: 1
- Medium drift items:   1
```

---

# 🧪 Local Testing

Check flattened output:

```sh
jq -r '
  paths(scalars) as $p |
  [
    ($p | map(if type=="number" then "[\(.|tostring)]" else tostring end) | join(".")),
    (getpath($p) | tostring)
  ] | @tsv
' file.json
```

Test workflow locally:

```sh
brew install act
act -j drift
```

---

# 🤝 Contributing

See `CONTRIBUTING.md`.  
Pull requests and enhancements welcome (profiles for WP Rocket, Terraform, etc.).

---

# 🛡 Security

Do not share production secrets or configuration in issues.  
See `SECURITY.md` for private disclosure instructions.

---

# 📄 License

MIT License.  
See `LICENSE` for details.
