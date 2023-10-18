![](/banner.png)

<h1 align="center">laravel-deploy-preview</h1>

<p align="center">
    <strong>A GitHub Action to create on-demand preview environments for Laravel apps.</strong>
</p>

<p align="center">
    <!-- TODO test status -->
    <a href="https://github.com/TzviPM/laravel-deploy-preview/blob/main/LICENSE"><img src="https://img.shields.io/badge/license-MIT-darkcyan.svg" alt="MIT License"></a>
</p>

## About

`TzviPM/laravel-deploy-preview` is a GitHub Action to automatically deploy new Laravel app instances to Laravel Forge using Envoyer and branch databases using PlanetScale. It's designed for creating PR preview environments that are isolated, publicly accessible (or privately, depending on your server's settings), and closely resemble your production environment, to preview and test your changes.

When you open a PR and this action runs for the first time, it will:

- Create a new site on Forge with a unique subdomain and install your Laravel app into it.
- Create a new project in Envoyer with your PR branch and link it to the site in Forge.
- Create a new database branch in PlanetScale and configure your app to use it.
- Create and install an SSL certificate and comment on your PR with a link to the site.
- Set up a scheduled job in Forge to run your site's scheduler.
- Enable [Deploy When Code Is Pushed](https://docs.envoyer.io/projects/management.html#source-control) on the site so that it updates automatically when you push new code.

## Requirements

Before adding this action to your workflows, make sure you have:

- A Laravel Forge [app server](https://forge.laravel.com/docs/1.0/servers/types.html#app-servers).
- An Envoyer account (https://envoyer.io).
- A PlanetScale Database (https://planetscale.com/).
- A [wildcard subdomain DNS record](https://en.wikipedia.org/wiki/Wildcard_DNS_record) pointing to your Forge server.
- A Forge [API token](https://forge.laravel.com/docs/1.0/accounts/api.html#create-api-token).
- An Envoyer API Token
- A PlanetScale [Service Token](https://api-docs.planetscale.com/reference/service-tokens#creating-a-service-token)

## Usage

> **Warning**: This action has direct access to your Laravel Forge account and should only be used in trusted contexts. Anyone who can push to a GitHub repository using this action will be able to execute code on the connected Forge servers.

Add your API tokens as an [Actions Secret](https://docs.github.com/en/actions/security-guides/encrypted-secrets#creating-encrypted-secrets-for-a-repository) in your GitHub repository. Then, use `TzviPM/laravel-deploy-preview` inside any workflow.

For the action to be able to clean up preview sites and other resources after a PR is merged, it has to be triggered on the pull request "closed" event. By default, GitHub's `pull_request` event does _not_ trigger a workflow run when its activity type is `closed`, so you may need to place this action in its own workflow file that specifies that event type:

```yaml
# deploy-preview.yml
on:
  pull_request:
    types: [opened, closed]
jobs:
  deploy-preview:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: TzviPM/laravel-deploy-preview@v1
        with:
          forge-token: ${{ secrets.FORGE_TOKEN }}
          pscale-token: ${{ secrets.PSCALE_TOKEN }}
          envoyer-token: ${{ secrets.ENVOYER_TOKEN }}
          servers: |
            qa-1.acme.dev 60041
```

### Inputs

#### `forge-token` (required)

The `forge-token` input parameter accepts your Forge API token, which the action uses to communicate with Laravel Forge to create sites and other resources. **Store this value in an encrypted secret; do not paste it directly into your workflow file.**

#### `pscale-token` (required)

The `pscale-token` input parameter accepts your PlanetScale service token, which the action uses to communicate with PlanetScale to create database branches. **Store this value in an encrypted secret; do not paste it directly into your workflow file.**

#### `envoyer-token` (required)

The `envoyer-token` input parameter accepts your Envoyer API token, which the action uses to communicate with Envoyer to create and manage projects. **Store this value in an encrypted secret; do not paste it directly into your workflow file.**

#### `servers` (required)

The `servers` input parameter accepts a list of Forge servers to deploy to.

Each server must include both a domain name and a server ID, separated by a space. The domain name should be the wildcard subdomain pointing at that server (without the wildcard part). For example, if your wildcard subdomain is `*.qa-1.acme.dev` and your Forge server ID is `60041`, set this input parameter to `qa-1.acme.dev 60041`.

If this input parameter contains multiple lines, each line will be treated as a different Forge server. We plan to support deploying to whichever server has the fewest sites already running on it, but the action currently only deploys to one server; if you list multiple servers, it will use the first one.

#### `after-deploy`

The `after-deploy` input parameter allows you to append additional commands to be run after the Forge deploy script.

Example:

```yaml
- uses: TzviPM/laravel-deploy-preview@v1
  with:
    forge-token: ${{ secrets.FORGE_TOKEN }}
    pscale-token: ${{ secrets.PSCALE_TOKEN }}
    envoyer-token: ${{ secrets.ENVOYER_TOKEN }}
    servers: |
      qa-1.acme.dev 60041
    after-deploy: npm ci && npm run build
```

#### `environment`

The `environment` input parameter allows you to add and update environment variables in the preview site.

Example:

```yaml
- uses: TzviPM/laravel-deploy-preview@v1
  with:
    forge-token: ${{ secrets.FORGE_TOKEN }}
    pscale-token: ${{ secrets.PSCALE_TOKEN }}
    envoyer-token: ${{ secrets.ENVOYER_TOKEN }}
    servers: |
      qa-1.acme.dev 60041
    environment: |
      APP_ENV=preview
      TELESCOPE_ENABLED=false
```

You can also use github secrets here. For example:

```yaml
- uses: TzviPM/laravel-deploy-preview@v1
  with:
    forge-token: ${{ secrets.FORGE_TOKEN }}
    pscale-token: ${{ secrets.PSCALE_TOKEN }}
    envoyer-token: ${{ secrets.ENVOYER_TOKEN }}
    servers: |
      qa-1.acme.dev 60041
    environment: |
      APP_ENV=preview
      TELESCOPE_ENABLED=false
      SOME_API_KEY=${{ secrets.SOME_API_PREVIEW_KEY }}
```

## Development

This action is based on [bakerkretzmar/laravel-deploy-preview](https://github.com/bakerkretzmar/laravel-deploy-preview). It's written in TypeScript and compiled with [`ncc`](https://github.com/vercel/ncc) into a single JavaScript file.

Run `npm run build` to compile a new version of the action for distribution.

To run the action locally, create a `.env` file and add your Forge API token to it, then edit `src/debug.ts` to manually set the input values you want to use, and finally run `npm run debug`.

When releasing a new version of the action, update the major version tag to point to the same commit as the latest patch release. This is what allows users to use `TzviPM/laravel-deploy-preview@v1` in their workflows instead of `TzviPM/laravel-deploy-preview@v1.0.2`. For example, after tagging and releasing `v1.0.2`, delete the `v1` tag locally, create it again pointing to the same commit as `v1.0.2`, and force push your tags with `git push -f --tags`.
