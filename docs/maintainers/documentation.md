# Deploy documentation

The site documents `main`. Documentation deployment uses `.github/workflows/docs.yml` and runs independently of package releases.

## Build and preview locally

```bash
uv sync --frozen
make docs-test
make docs
```

`make docs-test` builds with MkDocs strict mode. `make docs` starts a local server at `http://127.0.0.1:8000`. A successful build checks rendering and configured local links; run tutorial commands separately to validate their behavior.

The directories match the navigation: `tutorials/` contains the guided agent and client paths; `how-to/` contains specific tasks; `explanation/` covers concepts; `reference/` defines API and protocol contracts. Experimental contracts live under `reference/experimental/`. `integrations/` holds source studies, `migration/` holds version-specific upgrades, and `maintainers/` covers contributor operations.

When a page moves, add its old path to `redirect_maps` in `mkdocs.yml` and update links to the new path. The redirects plugin generates compatibility pages during the build, so old URLs do not require duplicate Markdown files in `docs/`. Keep existing fragment anchors at the destination where readers may have bookmarked them. Verify code against the SDK version the page documents, and distinguish repository examples from a default PyPI installation.

## Keep the existing Pages source

The repository publishes from the root of `gh-pages`. Keep Settings → Pages set to **Deploy from a branch**, with `gh-pages` and `/ (root)` selected. The Documentation workflow builds with MkDocs strict mode and commits the generated site to that branch using `mkdocs gh-deploy`.

The workflow explicitly requests a Pages build after pushing. GitHub notes that pushes made with `GITHUB_TOKEN` do not trigger a Pages build automatically; see [publishing-source troubleshooting](https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site#troubleshooting-publishing-from-a-branch). The build request uses the [Pages API](https://docs.github.com/en/rest/pages/pages#request-a-github-pages-build).

The `github-pages` environment must allow the `gh-pages` branch used by GitHub's Pages deployment. Package release tags do not publish documentation and do not need deployment permission. This setup uses the existing site and branch; no publishing-source switch is required.

## Publish and recover

The Main workflow checks documentation on pull requests. The Documentation workflow runs when relevant files change on `main`, including documentation, source, examples, build configuration, and dependencies. It fetches the existing `gh-pages` history, builds strictly, and pushes a new site commit without a force push.

Only the main-only publication job receives `contents: write` to update `gh-pages` and `pages: write` to request its build. PR checks use read-only permissions. The Pages concurrency group serializes publication runs without cancelling an in-progress run.

To rebuild without another source commit, select **Documentation** in Actions and run it with `main` selected. Runs on other refs skip publication. This operation does not publish a Python package and does not require `PYPI_TOKEN` or a separate deployment token.

After publication, check the Documentation run and GitHub's Pages build/deployment run, then verify the [site](https://agentclientprotocol.github.io/python-sdk/) shows the expected changes. A successful push to `gh-pages` alone is not proof that Pages has finished deploying.
