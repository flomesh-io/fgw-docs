# Flomesh Service Mesh Docs

:book: This section contains the [FGW Docs](https://github.com/flomesh-io/fgw-docs)

:link: Looking for the main FGW website? Visit [FGW](https://github.com/flomesh-io/fgw)


## Editing Content

fgw-docs.flomesh.io is a static site. The documentation content needs to be located at `content/docs/`.

The content served on [https://fgw-docs.flomesh.io](https://fgw-docs.flomesh.io) is served from the latest release on this repo. Most updates should be made only in main and can be previewed at [https://fgw-docs.flomesh.io/](https://fgw-docs.flomesh.io/).

If it's necessary to change published release-specific docs, those changes should be made in the release-specific branch serving those docs. Once configured as described in [Adding release-specific docs](#adding-release-specific-docs), PRs to that branch will auto-build just like PRs to main.

References to the fgw branch, fgw version, and pipy version should be parameterized unless they are in a sample output. 

To ensure the docs content renders correctly in the theme, each page will need to have [front matter](https://gohugo.io/content-management/front-matter/) metadata. Example front matter:

```
---
title: "Docs Home"
linkTitle: "Home"
description: "FGW Docs Home"
weight: 1
type: docs
---
```

## Front Matter Notes:

* inclusion of `type: docs` is important for the theme to properly index the site contents
* the `linkTitle` attribute allows you to simplify the name as it appears in the left-side nav bar - ideally it should be short and clear - whereas the title can handle long form names for pages/documents.

## Adding release-specific docs

### Create a release branch

Look for a branch in the upstream repo named `release-vX.Y`, where `X` and `Y` correspond to the major and minor version of the new release. For example, [release-v0.8](https://github.com/flomesh-io/fgw-docs/tree/release-v1.0). If the branch already exists, move to the next step.

Identify the base commit in the main branch for the release and cut a release branch off main.

> Note: Care must be taken to ensure the release branch is created from a commit meant for the release. If unsure about the commit to use to create the release branch, please open an issue in the fgw repo and a maintainer will assist you with this.

```console
$ git checkout -b release-<version> <commit-id> # ex: git checkout -b release-v1.1 0d05587
```

Push the release branch to the upstream repo (NOT forked), identified here by the upstream remote.

```console
$ git push upstream release-<version> # ex: git push upstream release-v1.4
```


### Update the release references

Proceed with the following steps once the release branch has been created in the FGW repo.

1. Create a new branch off of the release branch to maintain updates specific to the new version. Let's call it the patch branch. The patch branch should not be created in the upstream repo.
2. On the patch branch, update the `fgw_branch`, `fgw_version`, and `envoy_version` in [config.toml](https://github.com/flomesh-io/fgw-docs/blob/main/config.toml) to the new release versions.

    ```toml
    fgw_branch = "release-v0.8"
    fgw_version = "v1.9.0"
    pipy_version = "0.90.1-87"
    ```
3. Create a pull request from the patch branch to the release branch. Proceed to the next step once the pull request is approved and merged.

### After cutting the FGW release

1. Open an issue in this repo asking for a new DNS record be added to the site (via Netlify), to assign a subdomain to the deployed branch.
2. When published, the newly-added branch will function like [https://fgw-docs.flomesh.io/](https://rfgw-docs.flomesh.io/)

### After publishing the new release-specific docs

#### Update the release branch

1. Create another patch branch off of the release branch (or use the existing patch branch after fetching and rebasing on the release branch).
2. On the patch branch, add the new release as a version parameter in [config.toml](https://github.com/flomesh-io/fgw-docs/blob/main/config.toml) with the `latest` tag. The version parameters represent all currently supported versions of FGW and are used to populate the Release drop down menu on the site. The url for the new release is formatted `https://release-vX-Y.docs.openservicemesh.io/`.

    ```toml
    [[params.versions]]
        version = "v0.8 (latest)"
        url = "https://release-v0-8.docs.openservicemesh.io/"
    ```

3. Create a pull request from the patch branch to the release branch. Proceed to the next step once the pull request is approved and merged.

#### Update main and previous release branches

1. Update the `redirects` in `netlify.toml` on the main branch to redirect `https://fgw-docs.flomesh.io/` to `https//release-vX-Y.docs.openservicemesh.io/` where `release-vX-Y` is the newest release.
2. Each previous release-specific site that is still supported needs to be able to access the latest release from the Release drop down. On the previous release branches, update the [config.toml](https://github.com/flomesh-io/fgw-docs/blob/main/config.toml) to list the new release version as shown above.
3. The `latest` tag must be removed from all previous versions. For example, the `latest` tag must be removed from `v0.7 (latest)` on the `release-v0.7` branch.
4. Update the version banner parameter in `config.toml` to enable the banner at the top of each previous release-specific site that will tell visitors which version they are looking at. For example, the version banner for `release-v1-1` would be configured as follows:
    ```toml
    [params.versionbanner]
	    show = true
	    archive = "v1.1"
	    latest = "https://release-v1-1.fgw-docs.flomesh.io/"
    ```
5. Update `content/docs/releases/docs.md` to include the new release and update the inactive releases list.
6. 
### Update the release support matrix

Once a new FGW version has been released, update the [FGW and Kubernetes support matrix](./content/docs/guides/install.md#kubernetes-support). The Kubernetes version support will be the [current releases of Kubernetes](https://kubernetes.io/releases/) at the time of the FGW release.

### Copy files for localization

Copy entire directory structure under `content/en` to each other localization directory, such as `content/zh`. Each file in each localization must be individually translated for a given release. For more details on the translation process, see [Localization](#localization).

## Localization

The FGW docs can be translated into multiple localizations. To ensure the most accurate content, each localization must be translated for every release and each file should be manually translated for a given localization. When a new release is published, the English version of every content file for that release is copied to each localization. Those temporary English files can then be individually translated.

**If you find a technical error in the content, file a PR to the English version of the content file in the `main` branch so that the fix is carried forward into future releases. Localized content for a release is not carried forward into the next release so any fixes applied directly to localized content will be lost.**

### Localizing a file

To localize one or more files:

- Create a new branch off of the latest release branch, such as `release-v1.1`.
- Translate the temporary English version of the file(s) in the existing localization directory, such as `content/zh/_index.md`.
- File a PR targeting the latest release branch.

### Adding a new localization

When you add a new localization, you need to create *BOTH* an initial translation for the latest release as well as the scaffold for the new localization in the `main` branch. 

To create the initial translation for the latest release:

- Create a branch based on the latest release branch, such as `release-v1.1`.
- Add a new localization directory to the `content/` directory.
- Copy the entire directory structure under `content/en` to the new localization directory.
- Add a new language to `config.toml` in the `[languages]` section.
- Optionally, you can begin translating one or more files in the new localization directory.
- File a PR targeting the latest release branch.

To create the scaffold for the new localization:

- Create a branch based on `main`.
- Add the same localization directory to the `content/` directory, but only containing a `.gitkeep` file.
- Copy the new language configuration to `config.toml` in the `[languages]` section, but comment it out.
- File a PR targeting the `main` branch.

# Site Development

## Notes

* built with the [Hugo](https://gohugo.io/) static site generator
* custom theme uses [Docsy](https://www.docsy.dev/) as a base, with [Bootstrap](https://getbootstrap.com/) as the underlying css framework.
* deployed to [Netlify](https://app.netlify.com/sites/fgw-docs/deploys) via merges to main. (@flynnduism can grant additional access to account)
* metrics tracked via Google Analytics

## Install dependencies:

* Hugo [installation guide](https://gohugo.io/getting-started/installing/)
* NPM packages are installed by running `yarn`. [Install Yarn](https://yarnpkg.com/getting-started/install) if you need to.

## Run the site:

```
// install npm packages
yarn

// rebuild the site (to compile latest css/js)
hugo

// or serve the site for local dev
hugo serve
```

## Deploying the site:

`hugo serve` will run the site locally at [localhost:1313](http://localhost:1313/)
