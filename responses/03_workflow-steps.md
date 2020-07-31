# Workflow steps

So far, the workflow knows what the trigger is and what environment to run in. But, what exactly is supposed to run? The "steps" section of this workflow specifies actions and scripts to be run in the Ubuntu environment when new labels are added.

## Step 3: Set up the environment for your app

We won't be going into detail on the steps of this workflow, but it would be a good idea to become familiar with the actions we're using. They are:

- [`actions/checkout`](https://github.com/actions/checkout)
- [`actions/upload-artifact`](https://github.com/actions/upload-artifact)
- [`actions/download-artifact`](https://github.com/actions/download-artifact)
- [`azure/webapps-container-deploy`](https://github.com/Azure/webapps-container-deploy)

The course [_Using GitHub Actions for CI_](https://lab.github.com/githubtraining/github-actions:-continuous-integration) also teaches how to use most of these actions in details.

### :keyboard: Activity: Store your credentials in GitHub secrets and finish setting up your workflow
1. In a new tab, [create an Azure account](https://azure.microsoft.com/en-us/free/) if you don't already have one. If your Azure account is created through work, you may encounter issues accessing the necessary resources -- we recommend creating a new account for personal use and for this course. 
    > Note: You may need a credit card to create an Azure account. If you're a student, you may also be able to take advantage of the [Student Developer Pack](https://education.github.com/pack) for access to Azure. If you'd like to continue with the course without an Azure account, Learning Lab will still respond, but none of the deployments will work.
1. Create a [new subscription](https://docs.microsoft.com/en-us/azure/cost-management-billing/manage/create-subscription) in the Azure Portal. 
    > Note: your subscription must be configured "Pay as you go" which will require you to enter billing information. This course will only use a few minutes from your free plan, but Azure requires the billing information. 
1. Install [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest) on your machine. 
1. In your terminal, run:
    ```shell
    az login
    ```
1. Copy the value of the `id:` field to a safe place. We'll call this `AZURE_SUBSCRIPTION_ID`. Here's an example of what it looks like:
    ```shell
    [
    {
      "cloudName": "AzureCloud",
      "id": "f****a09-****-4d1c-98**-f**********c", # <-- Copy this id field
      "isDefault": true,
      "name": "some-subscription-name",
      "state": "Enabled",
      "tenantId": "********-a**c-44**-**25-62*******61",
      "user": {
        "name": "mdavis******@*********.com",
        "type": "user"
        }
      }
    ]
    ```
1. In your terminal, run the command below. **Note: The `\` character works as a line break on Unix based systems.  If you are on a Windows based system the `\` character will cause this command to fail.  Place this command on a single line if you are using Windows.**
    ```shell
    az ad sp create-for-rbac --name "GitHub-Actions" --role contributor \
                              --scopes /subscriptions/{subscription-id} \
                              --sdk-auth
                              
    # Replace {subscription-id} with the same id stored in AZURE_SUBSCRIPTION_ID.
    ```   
1. Copy the entire contents of the command's response, we'll call this `AZURE_CREDENTIALS`. Here's an example of what it looks like:
    ```shell
    {
      "clientId": "<GUID>",
      "clientSecret": "<GUID>",
      "subscriptionId": "<GUID>",
      "tenantId": "<GUID>",
      (...)
    } 
    ```
1. Back on GitHub, click on this repository's **[Secrets]({{ repoUrl }}/settings/secrets)** in the Settings tab.
1. Click **New secret**
1. Name your new secret **AZURE_SUBSCRIPTION_ID** and paste the value from the `id:` field in the first command.
1. Click **Add secret**.
1. Click **New secret** again.
1. Name the second secret **AZURE_CREDENTIALS** and paste the entire contents from the second terminal command you entered.
1. Click **Add secret**
1. Back in this pull request, edit the `.github/workflows/deploy-staging.yml` file to use some new actions, or [use this quick link]({{ repoUrl }}/edit/staging-workflow/.github/workflows/deploy-staging.yml?) _(We recommend opening the quick link in another tab)_

If you'd like to copy the full workflow file, it should look like this:

```yml
name: Stage the app

on: 
  pull_request:
    types: [labeled]

env:
  DOCKER_IMAGE_NAME: {{user.login}}-azure-ttt
  IMAGE_REGISTRY_URL: docker.pkg.github.com
  #################################################
  ### USER PROVIDED VALUES ARE REQUIRED BELOW   ###
  #################################################
  #################################################
  ### REPLACE USERNAME WITH GH USERNAME         ###
  AZURE_WEBAPP_NAME: {{user.login}}-ttt-app
  #################################################

jobs:
  build:
    if: contains(github.event.pull_request.labels.*.name, 'stage')

    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v1
      - name: npm install and build webpack
        run: |
          npm install
          npm run build
      - uses: actions/upload-artifact@master
        with:
          name: webpack artifacts
          path: public/

  Build-Docker-Image:
    runs-on: ubuntu-latest
    needs: build
    name: Build image and store in GitHub Packages
    steps:
      - name: Checkout
        uses: actions/checkout@v1

      - name: Download built artifact
        uses: actions/download-artifact@master
        with:
          name: webpack artifacts
          path: public

      - name: create image and store in Packages
        uses: mattdavis0351/actions/docker-gpr@1.3.0
        with:
          repo-token: {% raw %}${{secrets.GITHUB_TOKEN}}{% endraw %}
          image-name: {% raw %}${{env.DOCKER_IMAGE_NAME}}{% endraw %}

  Deploy-to-Azure:
    runs-on: ubuntu-latest
    needs: Build-Docker-Image
    name: Deploy app container to Azure
    steps:
      - name: "Login via Azure CLI"
        uses: azure/login@v1
        with:
          creds: {% raw %}${{ secrets.AZURE_CREDENTIALS }}{% endraw %}

      - uses: azure/docker-login@v1
        with:
          login-server: {% raw %}${{env.IMAGE_REGISTRY_URL}}{% endraw %}
          username: {% raw %}${{ github.actor }}{% endraw %}
          password: {% raw %}${{ secrets.GITHUB_TOKEN }}{% endraw %}

      - name: Deploy web app container
        uses: azure/webapps-deploy@v2
        with:
          app-name: {% raw %}${{env.AZURE_WEBAPP_NAME}}{% endraw %}
          images: {% raw %}${{env.IMAGE_REGISTRY_URL}}/${{ github.repository }}/${{env.DOCKER_IMAGE_NAME}}:${{ github.sha }}{% endraw %}

      - name: Azure logout
        run: |
          az logout
```

---

I'll respond when you push a commit on this branch.
