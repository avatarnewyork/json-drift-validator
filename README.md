# JSON Drift Validator

A GitHub Action that **detects configuration drift** between two JSON files — even deeply nested structures — using flattened key paths and wildcard-based severity rules.

Perfect for validating:

- WordPress configuration exports (WP Rocket, ACF, WooCommerce, etc.)
- Terraform JSON plan/state diffs
- `package.json` dependency drift
- CloudFormation templates
- Kubernetes flattened JSON manifests
- SaaS platform config exports (Auth0, Firebase, Algolia, etc.)
- Any JSON-based configuration

---

## 🌟 Features

- 🔍 **Flattened JSON comparison** — Nested JSON becomes dot-paths like `dependencies.react`  
- 🎯 **Wildcard severity rules** (`cache.*`, `terraform.*.versioning`, etc.)
- 🚫 **Ignore overrides critical & medium**  
- 🟥 **Critical drift → fail the workflow**
- 🟨 **Medium drift → warn but continue**
- 🧩 **Works with any JSON input**
- 📦 **Great for config-as-code workflows**

---

## 📥 Inputs

| Name | Required | Description |
|------|----------|-------------|
| `baseline_json` | Yes | The “known good” config JSON (usually a GitHub secret) |
| `compare_json` | Yes | The JSON to validate against the baseline |
| `critical_keys` | No | Comma-separated wildcard patterns that *must not* drift |
| `medium_keys` | No | Comma-separated wildcard patterns that warn on drift |
| `ignore_keys` | No | Comma-separated wildcard patterns that override and ignore drift |

---

## 📤 Outputs

| Name | Description |
|------|-------------|
| `critical_fail` | `1` if critical drift detected |
| `medium_warn` | `1` if medium drift detected |

A detailed drift report is uploaded as the artifact **`json-drift-report`**.

---

## 🚀 Example Usage

```yaml
jobs:
  validate:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Validate JSON drift
        id: drift
        uses: avatarnewyork/json-drift-validator@v1
        with:
          baseline_json: ${{ secrets.MY_BASELINE }}
          compare_json: ${{ steps.generate.outputs.config }}

          critical_keys: "dependencies.react,build.target,cache.*"
          medium_keys: "version,metadata.build_version"
          ignore_keys: "timestamps.*,logs.*,**.generatedAt"

      - name: Fail if critical drift detected
        if: steps.drift.outputs.critical_fail == '1'
        run: exit 1
