# Azure configuration

GitHub Actions is cloud agnostic, so any cloud will work. We'll show how to deploy to Azure in this course.

### Azure resources

We'll use the following Azure resources in this course:
- A [web app](https://docs.microsoft.com/en-us/azure/app-service/overview) is how we'll be deploying our application to Azure.
- A [resource group](https://docs.microsoft.com/en-us/azure/azure-resource-manager/management/overview#resource-groups) is a collection of resources, like web apps and virtual machines (VMs).
- An [App Service plan](https://docs.microsoft.com/en-us/azure/app-service/overview-hosting-plans) is what runs our web app and manages the billing (our app should run for free).

Through the power of GitHub Actions, we can create, configure, and destroy these resources through our workflow files. 

## Step 5: Spin up, configure, and destroy Azure resources

### :keyboard: Activity: Use your workflow file to configure your cloud resources

First, we'll set up a personal access token (PAT) that can be used with GitHub Packages. Then, we'll create a GitHub Actions workflow that automates the set up of resources in Azure, including the retrieval of a Docker image from Github Packages.

1. Navigate to your [**Personal Access Tokens**]({{ GITHUB_URL }}/settings/tokens).
2. Click on **Generate new token**.
3. Select the following scopes:
    - `write:packages`
    - `read:packages`
    - `delete:packages`
4. Click **Generate token** and store the token generated in a safe place. 
5. Back on this repository, access the repository's **[Secrets]({{ repoUrl }}/settings/secrets)** in the Settings tab.
6. Click **New secret**
7. Name your new secret **PACKAGE_PAT** and paste the value of your newly created token.
8. Click **Add secret**.
9.  Edit the `.github/CHANGETHIS/spinup-destroy.yml` file on this branch, or [use this quick link]({{ repoUrl }}/edit/azure-configuration/.github/CHANGETHIS/spinup-destroy.yml?). _(We recommend opening the quick link in another tab.)_
10. Rename the file to `.github/workflows/spinup-destroy.yml`
11. Change the value of the `AZURE_WEBAPP_NAME:` to `{{ user.login }}-ttt-app`
12. Commit your changes.

The file should look like this:

```yaml
name: Configure Azure environment

on: 
  pull_request:
    types: [labeled]

env:
  AZURE_RESOURCE_GROUP: cd-with-actions
  AZURE_APP_PLAN: actions-ttt-deployment
  AZURE_LOCATION: '"Central US"'
  #################################################
  ### USER PROVIDED VALUES ARE REQUIRED BELOW   ###
  #################################################
  #################################################
  ### REPLACE USERNAME WITH GH USERNAME         ###
  AZURE_WEBAPP_NAME: {{user.login}}-ttt-app
  #################################################

jobs:
  setup-up-azure-resources:
    runs-on: ubuntu-latest

    if: contains(github.event.pull_request.labels.*.name, 'spin up environment')

    steps:
      - name: Checkout repository
        uses: actions/checkout@v2

      - name: Azure login
        uses: azure/login@v1
        with:
          creds: {% raw %}${{ secrets.AZURE_CREDENTIALS }}{% endraw %}

      - name: Create Azure resource group
        if: success()
        run: |
          {% raw %}az group create --location ${{env.AZURE_LOCATION}} --name ${{env.AZURE_RESOURCE_GROUP}} --subscription ${{secrets.AZURE_SUBSCRIPTION_ID}}{% endraw %}

      - name: Create Azure app service plan
        if: success()
        run: |
          {% raw %}az appservice plan create --resource-group ${{env.AZURE_RESOURCE_GROUP}} --name ${{env.AZURE_APP_PLAN}} --is-linux --sku F1 --subscription ${{secrets.AZURE_SUBSCRIPTION_ID}}{% endraw %}

      - name: Create webapp resource
        if: success()
        run: |
          {% raw %}az webapp create --resource-group ${{ env.AZURE_RESOURCE_GROUP }} --plan ${{ env.AZURE_APP_PLAN }} --name ${{ env.AZURE_WEBAPP_NAME }}  --deployment-container-image-name nginx --subscription ${{secrets.AZURE_SUBSCRIPTION_ID}}{% endraw %}

      - name: Configure webapp to use GitHub Packages
        if: success()
        run: |
          {% raw %}az webapp config container set --docker-custom-image-name nginx --docker-registry-server-password ${{secrets.PACKAGE_PAT}} --docker-registry-server-url https://docker.pkg.github.com --docker-registry-server-user ${{github.actor}} --name ${{ env.AZURE_WEBAPP_NAME }} --resource-group ${{ env.AZURE_RESOURCE_GROUP }} --subscription ${{secrets.AZURE_SUBSCRIPTION_ID}}{% endraw %}

  destroy-azure-resources:
    runs-on: ubuntu-latest

    if: contains(github.event.pull_request.labels.*.name, 'destroy environment')

    steps:
      - name: Checkout repository
        uses: actions/checkout@v2

      - name: Azure login
        uses: azure/login@v1
        with:
          creds: {% raw %}${{ secrets.AZURE_CREDENTIALS }}{% endraw %}

      - name: Destroy Azure environment
        if: success()
        run: |
          {% raw %}az group delete --name ${{env.AZURE_RESOURCE_GROUP}} --subscription ${{secrets.AZURE_SUBSCRIPTION_ID}} --yes{% endraw %}
```

---

I'll respond when I detect a commit on this branch.
