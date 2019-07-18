---
date: 2018-07-18
title: Using Github Artifacts as a Pipeline Trigger
categories:
   - Pipelines
description: Using Github Artifacts as a Pipeline Trigger
type: Document
---

1. Enable the Artifacts feature in `config/spinnaker-local.yml`

    ```yaml
    features:
      artifacts:
        enabled: true
    ```
2. Configure a Github Artifact account in `config/clouddriver-local.yml`. You'll need a [Github Personal Access Token](https://blog.github.com/2013-05-16-personal-api-tokens/) so that Clouddriver can fetch artifacts stored in your repositories. Most users have a bot account that they can use for automation like this. If you already have one, feel free to use it instead of creating a new one. `my-github-account` can be changed as you see fit. If you have multiple Github artifact accounts, it's best to use an identifiable name.
    ```yaml
    artifacts:
      github:
        enabled: true
        accounts:
        - name: my-github-account # this will be the name of the artifact account within Spinnaker
          username: <YOUR_ACCOUNT_USERNAME>
          token: <YOUR_PERSONAL_ACCESS_TOKEN>
    ```

3. Redeploy Armory Spinnaker

4. Configure a Github webhook using [this documentation](https://www.spinnaker.io/setup/triggers/github/). If you have a lot of repositories, you can setup an Organization level webhook.

5. Configure your Github artifacts and pipeline triggers following [this documentation](https://www.spinnaker.io/guides/user/pipeline/triggers/github/#using-github-artifacts-in-pipelines).

***

### More Resources: 
- [About Artifacts in Spinnaker](https://www.spinnaker.io/reference/artifacts/)
- [Github Artifact Type](https://www.spinnaker.io/reference/artifacts/types/github-file/)
- [Github Personal Access Tokens](https://blog.github.com/2013-05-16-personal-api-tokens/)
- [Other Artifact account types](https://docs.armory.io/install-guide/adding_accounts/#adding-artifact-accounts)