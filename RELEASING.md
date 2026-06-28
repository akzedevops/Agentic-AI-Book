# Building & Releasing the PDF

This repo turns the Markdown chapters in `book/chapters/` into a single PDF
(`agentic-ai-free-guide-v0.1.pdf`). The build is automated by the GitHub
Actions workflow at [`.github/workflows/build-pdf.yml`](.github/workflows/build-pdf.yml),
which renders the chapters with headless Chromium (via Playwright) using the
Noto Myanmar fonts so the text shapes correctly.

You almost never run the build by hand — CI does it for you. This document
explains what happens automatically and how to cut a release.

## The mental model

| You do | `Build PDF` job | `Attach PDF to Release` job |
| --- | --- | --- |
| Open / update a pull request | ✅ runs, uploads artifact | skipped |
| Merge to `main` | ✅ runs, uploads artifact | skipped |
| Push a `v*` tag | ✅ runs | ✅ **publishes a Release** |

- **Artifacts** are work-in-progress previews. They live inside the Actions run
  (30-day retention) and are only visible to people who open that run.
- **Releases** are permanent, public, and versioned. They are created on
  purpose, by pushing a version tag.

The release job is gated by `if: startsWith(github.ref, 'refs/tags/v')`, which
is why it is *skipped* on every PR and every push to `main`, and only *runs*
when the trigger is a `v*` tag.

## Everyday editing flow

Work on a branch, never directly on `main`:

```bash
git checkout main && git pull
git checkout -b edit/ch07-fixes
# ...edit book/chapters/*.md...
git commit -am "docs: fix ch7 wording"
git push -u origin edit/ch07-fixes
gh pr create
```

Then:

1. **PR opened/updated** → the `Build PDF` job runs. Wait for the green check,
   then open the run and download the PDF from **Artifacts** to eyeball it
   before merging.
2. **PR merged to `main`** → `Build PDF` runs again on `main`, confirming the
   main branch always builds and producing a fresh artifact.

No tags, no releases in this loop — that is 95% of the work.

## Cutting a release

When `main` is in a state worth publishing:

```bash
git checkout main && git pull          # release exactly what's on main
git tag -a v0.2 -m "v0.2 review build"
git push origin v0.2
```

Pushing the tag triggers the workflow with `github.ref = refs/tags/v0.2`, so
**both** jobs run: the PDF is built, then `Attach PDF to Release` publishes a
GitHub Release with the PDF attached and auto-generated release notes.

Watch it finish and confirm the asset:

```bash
gh run watch --exit-status
gh release view v0.2
```

## Conventions

- **Tag `main`, not a feature branch.** Tag the commit *after* it is merged so
  the release reflects exactly what is on `main`.
- **Use ordered tags** so releases sort correctly: `v0.1` → `v0.2` → `v1.0`.
- **Never move or reuse a tag** that has already been released. Cut a new one.

## Fixing a bad release

If a release comes out wrong, delete the release and tag, then re-tag:

```bash
gh release delete v0.2 --yes
git push origin :refs/tags/v0.2
git tag -d v0.2
# ...fix on main, then tag again...
```

## Building locally (optional)

CI is the source of truth, but you can reproduce a build on your machine:

```bash
pip install -r requirements-pdf.txt
python -m playwright install chromium
python scripts/build_book_pdf.py
# output: output/pdf/agentic-ai-free-guide-v0.1.pdf
```

For Myanmar glyphs to render instead of tofu boxes, you need the Noto Myanmar
fonts installed locally (e.g. `fonts-noto-core` on Debian/Ubuntu). The `output/`
directory is gitignored, so locally built PDFs are never committed.
