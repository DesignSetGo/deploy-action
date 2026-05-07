# DesignSetGo Apps — Deploy Action

A GitHub Action that deploys a [DesignSetGo Apps](https://github.com/DesignSetGo/designsetgo-apps)
bundle to a WordPress site on push.

## Quick start

1. **Generate a WordPress Application Password** for a user with the
   `manage_options` capability: `wp-admin → Users → Profile → Application Passwords`.

2. **Add three secrets** to your GitHub repo (Settings → Secrets and variables → Actions):

   - `DSGO_SITE` — your site URL (e.g. `https://example.com`)
   - `DSGO_USER` — your WordPress username
   - `DSGO_APP_PASSWORD` — the Application Password from step 1

3. **Add a workflow file** at `.github/workflows/deploy.yml`:

   ```yaml
   name: Deploy DSGo App
   on:
     push:
       branches: [main]

   concurrency:
     group: dsgo-deploy-${{ github.ref }}
     cancel-in-progress: true

   jobs:
     deploy:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v4
         - uses: actions/setup-node@v4
           with:
             node-version: '20'
         - run: npm ci && npm run build  # if your bundle has a build step
         - uses: DesignSetGo/deploy-action@v1
           with:
             site: ${{ secrets.DSGO_SITE }}
             username: ${{ secrets.DSGO_USER }}
             app-password: ${{ secrets.DSGO_APP_PASSWORD }}
             bundle: dist
   ```

4. **Push to main.** Your app deploys to `https://<site>/apps/<id>`.

## Inputs

| Input          | Required | Default | Description |
|----------------|----------|---------|-------------|
| `site`         | yes      | —       | WordPress site URL |
| `username`     | yes      | —       | WordPress username (must have `manage_options`) |
| `app-password` | yes      | —       | WordPress Application Password |
| `bundle`       | no       | `''`    | Path to the bundle directory. Empty = `./dist` if it exists, else repo root. |
| `cli-version`  | no       | `^1`    | npm semver range for `@designsetgo/cli` |

## Outputs

| Output    | Description |
|-----------|-------------|
| `app-url` | URL of the deployed app |
| `app-id`  | Manifest id of the deployed app |

Use them in downstream steps:

```yaml
- uses: DesignSetGo/deploy-action@v1
  id: dsgo
  with:
    # ...
- run: echo "Deployed to ${{ steps.dsgo.outputs.app-url }}"
```

## Rotating the Application Password

If your Application Password is compromised or expired:

1. In WP admin: Users → Profile → Application Passwords → Revoke the old password.
2. Generate a fresh one. Copy it (WordPress only shows it once).
3. In GitHub: Settings → Secrets and variables → Actions → Update `DSGO_APP_PASSWORD`.
4. Re-run the failed workflow.

Your WordPress account password is never used here, so rotating the Application
Password does not log you out anywhere.

## Self-hosted runners

The action requires `node` ≥ 20 and `jq` on `PATH`. Both are preinstalled on
GitHub-hosted `ubuntu-latest` runners. If you self-host, install them in your
runner image.

## Troubleshooting

- **`auth_failed`** — the Application Password was rejected. Verify
  `DSGO_USER` matches the user the password was generated for, and confirm the
  password hasn't been revoked.
- **`partial_env_credentials`** — only one of `DSGO_USER` / `DSGO_APP_PASSWORD`
  reached the CLI. Re-check that all three secrets are populated and named
  correctly.
- **`no_manifest`** — the bundle path you set doesn't contain a
  `dsgo-app.json` file. Check `bundle:` matches your build output directory.

## Versioning

Pinned to `@v1` (floating major). Patch and minor updates roll forward
automatically. To pin tighter:

```yaml
- uses: DesignSetGo/deploy-action@v1.2.0
```

## Continuous integration

The smoke test (`.github/workflows/smoke.yml`) currently runs on manual
dispatch only because it depends on a public release zip of the
DesignSetGo Apps plugin being installable into a fresh wp-now instance.
Once that prerequisite is met, the workflow's `on:` block will be
restored to `pull_request` + `push`. Until then, run smoke from the
Actions tab when you want to verify a change end-to-end against a
real WordPress site.

## License

MIT — see [LICENSE](LICENSE).
