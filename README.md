# vc-nuget-publisher

Pushes NuGet packages of the `vc-module-*` repositories to nuget.org with
[Trusted Publishing](https://learn.microsoft.com/en-us/nuget/nuget-org/trusted-publishing).
This is the only repository with a nuget.org policy, so a new module repository needs nothing on
nuget.org.

## Why a separate repository

nuget.org matches a policy against two claims of the GitHub OIDC token: `repository` and
`job_workflow_ref`, and both must name the policy's repository. A reusable workflow called from
another repository never passes, so a module can't push through a shared workflow in `.github`.
It can hand its packages to a workflow that runs **here**, in its own run, and that run passes
the one policy below.

The policy lets this workflow push any package of the owner, so write access to this
repository is kept to a small group, and the push runs only from `main` (see Setup).

## How it works

1. A module workflow (`module-ci.yml`, `publish-nugets.yml`, `module-release-hotfix.yml`)
   builds the packages and calls `VirtoCommerce/.github/actions/publish-nuget-central`.
2. The action uploads the `.nupkg`/`.snupkg` files as an artifact, dispatches
   `publish.yml` here with the module repository, run id and artifact id, and waits for it.
   A failed push fails the module run.
3. `publish.yml` checks the request and stops unless:
   - the repository is `VirtoCommerce/vc-module-*`;
   - the run is `module-ci.yml`, `publish-nugets.yml` or `module-release-hotfix.yml`, on any
     branch (as with the per-repo policies; the module workflow decides which branches publish);
   - the run was started by `push` or `workflow_dispatch` in that repository, not a fork;
   - the run is still in progress, so it is waiting for this publish;
   - the artifact was uploaded by that run.
4. It downloads the artifact, gets a short-lived key from nuget.org (`NuGet/login`) and runs
   `dotnet nuget push --skip-duplicate`.

## Setup

Do these steps in this order. The module templates point here from the release that ships
`publish-nuget-central`, so the repository must work before those templates are deployed.

1. Create `VirtoCommerce/vc-nuget-publisher` as a **public** repository and copy
   `.github/workflows/publish.yml` and this README into it. It must be public: the org is on
   GitHub Free, where a private repository gets no environments, environment secrets or
   deployment branch rules, and the steps below depend on them.
2. **Access:** give write access only to the people who maintain publishing. The account
   behind the modules' `REPO_TOKEN` also needs write access, because dispatching a workflow
   requires it.
3. **Branch protection on `main`:** pull request with at least one review, no force pushes,
   no deletions.
4. **GitHub App** that reads the modules' artifacts: create an app owned by the VirtoCommerce
   organization (Settings → Developer settings → GitHub Apps → New GitHub App), webhook off,
   repository permission **Actions: Read-only** and nothing else (Metadata: Read is added
   automatically). Install it on the organization with **All repositories**: private module
   repos and repos created later are then covered with no setup in them. Generate a private
   key. The workflow mints a token per publish that reaches only the requesting module repo,
   with Actions: read.
5. **Environment `nuget-publish`:** under Deployment branches and tags, select "Selected
   branches and tags" and add `main`. Add the app's credentials to it:
   - variable `NUGET_PUBLISHER_APP_CLIENT_ID` — the app's Client ID;
   - secret `NUGET_PUBLISHER_APP_PRIVATE_KEY` — the whole `.pem` file.
6. **nuget.org policy:** from a checkout of `.github`:

   ```bash
   cd scripts/nuget-trusted-publishers
   node add-nuget-trusted-publishers.mjs --repos vc-nuget-publisher --workflows publish.yml --environment nuget-publish
   ```

   If nuget.org shows the policy as temporary, run the first publish within 7 days, or refresh
   it with `--activate`.
7. **Try it** on one module before deploying the templates to all of them: run its
   `publish-nugets.yml` from a branch that has the new template, with `publishAlpha` on.
8. Release `.github` and deploy the module templates. After every module is on the new
   templates, the per-repo policies on nuget.org can be removed.

## Troubleshooting

- **"Get a token for the module repository" fails:** the app is not installed on the module
  repo (installation set to selected repositories), or the client ID / private key in
  `nuget-publish` is wrong.

- **Dispatch returned no run id:** the module's `REPO_TOKEN` can't dispatch here (no write
  access), or the API version header changed.
- **"Run is 'completed'":** the module run stopped waiting (cancelled or timed out) before this
  run checked it. Re-run the module job.
- **`NuGet/login` fails:** the policy is missing, expired (temporary for more than 7 days), or
  the job did not run in `nuget-publish` from `main`.
