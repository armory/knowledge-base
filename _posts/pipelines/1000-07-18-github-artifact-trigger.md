---
date: 2018-07-18
title: Using Github Artifacts as a Pipeline Trigger
categories:
   - pipelines
description: Using Github Artifacts as a Pipeline Trigger
type: Document
---

1. Enable the Artifacts feature in `config/spinnaker-local.yml`

    ```yaml
    features:
      artifacts:
        enabled: true
    ```
2. Configure a Github Artifact account in `config/clouddriver-local.yml`. You'll need a [Github Personal Access Token](https://blog.github.com/2013-05-16-personal-api-tokens/) so that Clouddriver can fetch artifacts stored in your repositories. Most users have a bot account that they can use for automation like this. If you already have one, feel free to use it instead of creating a new one.
    ```yaml
    artifacts:
      github:
        enabled: true
        accounts:
        - name: github-account-name # this will be the name of the artifact account within Spinnaker
          username: armory-bot
          token: personalAccessToken
    ```

3. Configure Github as an SCM source in `config/igor-local.yml`
    ```yaml
    github:
      baseUrl: "https://api.github.com"
      accessToken: personalAccessToken
      commitDisplayLength: 8
    ```

4. Redeploy Armory Spinnaker

5. Configure a Github webhook using [this documentation](https://www.spinnaker.io/setup/triggers/github/). If you have a lot of repositories, you can setup an Organization level webhook.

6. Configure your Github artifacts and pipeline triggers following [this documentation](https://www.spinnaker.io/guides/user/pipeline/triggers/github/#using-github-artifacts-in-pipelines).

***

### More Resources: 
- [About Artifacts in Spinnaker](https://www.spinnaker.io/reference/artifacts/)
- [Github Artifact Type](https://www.spinnaker.io/reference/artifacts/types/github-file/)
- [Github Personal Access Tokens](https://blog.github.com/2013-05-16-personal-api-tokens/)
- [Other Artifact account types](https://docs.armory.io/install-guide/adding_accounts/#adding-artifact-accounts)