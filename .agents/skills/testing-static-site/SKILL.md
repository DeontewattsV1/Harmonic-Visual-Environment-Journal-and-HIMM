---
name: testing-static-site
description: Test the Harmonic Visual Environment Journal static site end-to-end by opening it locally in Chrome. Use when verifying HTML, CSS, PDF links, or content changes.
---

# Testing the Static Site

## Prerequisites
- Chrome browser available
- Repo cloned locally with latest `main`

## How to Test

1. Open `index.html` in Chrome:
   ```
   google-chrome file:///path/to/repo/index.html
   ```

2. Verify these sections render:
   - **Header**: H1 "Harmonic Visual Environment Journal & HIMM" + subtitle
   - **Overview**: Mentions "Harmonic Intention Mapping Model (HIMM)"
   - **Journal**: PDF download link pointing to `assets/journal.pdf`
   - **Framework Pillars**: All 5 listed (Affirmation, Sound, Behavior, Evidence, Aligned Action)
   - **Footer**: Shows correct license (currently CC0-1.0, must match the LICENSE file)

3. Check browser console (F12 → Console) for zero errors

4. Click the PDF link — should open in Chrome's PDF viewer

5. Verify CSS styles are applied (centered layout, system font, accent-colored links, border separators)

## CI Workflows

- **`main.yml`** — Primary workflow. Validates required files exist (`index.html`, `assets/journal.pdf`, `assets/styles.css`, `README.md`), checks PDF header, lints HTML with `html-validate`, renders content markdown. Build/deploy jobs only run on push to `main` or `workflow_dispatch` (skipped on PRs). Uses `actions/upload-pages-artifact@v4`.
- **`blank.yml`** — Basic CI placeholder (echo only).
- **`gem-push.yml`** — Pre-existing broken workflow (Ruby 2.6.x + missing `.gemspec`). Might be removed in the future.
- **`summary.yml`** — AI-powered issue summarizer, triggered on new issues only. Uses environment variable indirection for untrusted input (issue title/body) to prevent script injection.

## Workflow Security Validation

When changes touch `.github/workflows/summary.yml`, verify the script injection fix is structurally correct:

```bash
python3 -c "
import yaml
with open('.github/workflows/summary.yml') as f:
    data = yaml.safe_load(f)
steps = data['jobs']['summary']['steps']

# Check untrusted input flows through env vars
prepare = next(s for s in steps if s.get('name') == 'Prepare issue text')
env = prepare.get('env', {})
assert 'ISSUE_TITLE' in env, 'ISSUE_TITLE must be in env'
assert 'ISSUE_BODY' in env, 'ISSUE_BODY must be in env'
assert 'dd if=/dev/urandom' in prepare.get('run',''), 'Random EOF marker required'

# Check inference step does NOT directly interpolate github.event
inference = next(s for s in steps if s.get('name') == 'Run AI inference')
prompt = inference.get('with',{}).get('prompt','')
assert 'env.ISSUE_PROMPT' in prompt, 'Prompt must use env var'
for s in steps:
    for v in s.get('with',{}).values():
        if isinstance(v,str):
            assert 'github.event.issue' not in v, f'Direct interpolation found in {s.get(\"name\")}'
print('ALL SECURITY CHECKS PASSED')
"
```

When changes touch `main.yml`, verify action versions:
- `actions/upload-pages-artifact` should be `@v4` (v3 is deprecated)
- `actions/checkout` should be `@v4`
- `actions/setup-node` should be `@v4`
- `actions/configure-pages` should be `@v5`
- `actions/deploy-pages` should be `@v4`

## Known Issues

- `assets/journal.pdf` is a placeholder — needs to be replaced with the real notebook.
- `_config.yml` and `assets/css/style.scss` are Jekyll safety nets for the auto-generated `pages-build-deployment` workflow. The `main.yml` workflow uses Node.js, not Jekyll. These files can be removed once GitHub Pages source is set to "GitHub Actions".
- The GitHub Pages deployment requires **Settings → Pages → Source** set to **"GitHub Actions"** (not "Deploy from a branch"). Without this, the auto-generated workflow may try to build from a `docs/` folder that doesn't exist.
- The `jekyll-docker.yml` / `pages-build-deployment` workflow may fail with Ruby 2.6.x errors on ubuntu-24.04. This is a pre-existing issue unrelated to any security fixes.
- Python's `yaml.safe_load` parses the YAML `on:` key as boolean `True`. When checking workflow structure programmatically, use `True in data` instead of `'on' in data`.

## Devin Secrets Needed
None required for local testing.
