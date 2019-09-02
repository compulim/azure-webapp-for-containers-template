# Azure Web App for Containers with Travis CI

> If you like this template, please [star us](https://github.com/compulim/azure-webapp-for-containers-template/stargazers).

This repository is a template with best practices for setting up a simple web server on Azure Web App using Docker single image and continuously deployed using GitHub and Travis CI.

We assume you have some understanding and experiences on these key technologies:

- Microsoft Azure
   - Azure CLI
   - Azure Container Registry
   - Azure Web Apps
   - Service Principal
- Docker
- GitHub
- Travis CI

## Why this template?

If you have hands-on experience on the technologies above, you are probably doing something very similar to the steps we outlined here. This tutorial will save you time for building your CI/CD scripts with best practices.

The CI/CD scripts outline here do not use Azure Container Registry webhooks feature, which would save you some time and costs.

Although this tutorial focus on Node.js, we are using it as a simple static web server. You are not required to run your app on Node.js base image.

## Why Docker?

Docker is a great tool for modern app deployment.

Developers can easily run a production server on their own box to verify the setup before rolling out. And the script-based approach make it easy to reproduce the image within minutes. This would greatly reduce development round-trip time and improve developer experience.

## Setup

### GitHub and Travis CI

1. Create a new GitHub repository based on this template
1. Enable Travis CI on the GitHub repository
   - Set environment variable `DOCKER_IMAGE_NAME` to `myapp-image`
   - Travis CI will also help you to test the pull request

### Microsoft Azure

There are multiple resources needed to create and configure.

- [Create resource group](#create-resource-group)
- [Create Azure Container Registry](#create-azure-container-registry)
- [Create App Service Plan for Linux](#create-app-service-plan-for-linux)
- [Create Azure Web App for Containers](#create-azure-web-app-for-containers)
- [Create service principal](#create-service-principal)

In this article, we will cover the setup using [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest).

#### Create resource group

Resource group groups all resources in a single place. We prefer to keep all resources in a single resource group to ease maintenance.

```sh
az group create \
  --name myapp-rg \
  --location "West US"
```

#### Create Azure Container Registry

Azure Container Registry keeps your Docker images.

```sh
az acr create \
  --resource-group myapp-rg \
  --name myappacr \
  --admin-enabled \
  --location "West US" \
  --sku Basic
```

> `--admin-enabled` turns on password-based access, which is required for deploying to Azure Web App.

After the registry is created, copy the name of the registry to Travis CI as environment variable named `ACR_NAME`.

Also, save the `id` value. We will use it in "[Create service principal](#create-service-principal)" step.

> If you lost the `id` value, you can run `az acr show --resource-group myapp-rg --name myappacr --query id --output tsv`.

#### Create App Service Plan for Linux

App Service Plan is the computational resources for your web app.

```sh
az appservice plan create \
  --resource-group myapp-rg \
  --name myapp-plan \
  --is-linux \
  --location "West US" \
  --sku B1
```

#### Create Azure Web App for Containers

Web App is a website hosted under the App Service Plan. One App Service Plan can host multiple Web App.

```sh
az webapp create \
  --resource-group myapp-rg \
  --name myapp-web \
  --deployment-container-image-name nginx \
  --plan myapp-plan
```

> Note: We will temporarily deploying NGINX image before our CI/CD pipeline is up.

After Azure Web App is created:

1. Verify it is running at https://myapp-web.azurewebsites.net/
1. Copy the name of the web app to Travis CI as environment variable named `AZURE_WEBAPP_NAME`
1. Copy the resource group of the web app to Travis CI as environment variable named `AZURE_WEBAPP_RESOURCE_GROUP`

Also, save the `id` value. We will use it in "[Create service principal](#create-service-principal)" step.

> If you lost the `id` value, you can run `az webapp show --resource-group myapp-rg --name myapp-web --query id --output tsv`.

#### Create service principal

Service principal is the service account to access your resources, a.k.a. service account. We will grant "Contributor" role to both Azure Container Registry (for reading admin password) and Azure Web App (for deployment).

> We found it is more stable to create service principal without assignments first, then assign role to it.

```sh
az ad sp create-for-rbac --name https://myapp-web --skip-assignment
```

You should see the result similar to below.

```json
{
  "appId": "12345678-1234-5678-abcd-12345678abcd",
  "displayName": "azure-cli-2018-12-25-12-34-56",
  "name": "https://myapp-web",
  "password": "1a2b3c4d-1a2b-3c4d-5e6f-1a2b3c4d5e6f",
  "tenant": "87654321-4321-8765-dcba-dbca87654321"
}
```

You should copy these values to Travis CI environment variables;

- Value of `appId` should be copied to `AZURE_SP_USERNAME`
- Value of `password` should be copied to `AZURE_SP_PASSWORD`
- Value of `tenant` should be copied to `AZURE_SP_TENANT`

Then, add assign "Contributor" role of this service principal the resources. In the following script, you shoudl replace `--assignee` with the `appId`, and `--scope` with the two saved `id` from ACR and Azure Web App respectively.

```sh
az role assignment create \
  --assignee 12345678-1234-5678-abcd-12345678abcd \
  --role Contributor \
  --scope /subscriptions/87654321-4321-8765-dcba-dbca87654321/resourceGroups/myapp-rg/providers/Microsoft.ContainerRegistry/registries/myappacr

az role assignment create \
  --assignee 12345678-1234-5678-abcd-12345678abcd \
  --role Contributor \
  --scope /subscriptions/87654321-4321-8765-dcba-dbca87654321/resourceGroups/myapp-rg/providers/Microsoft.Web/sites/myapp-web
```

### Kick off the build

For the first time, you will need to kick off the build manually.

Go to Travis CI of your repository, and then click "Trigger build". In about 2-3 minutes, you should see your website up and running at https://myapp-web.azurewebsites.net/. It should read "Hello, World!".

Subsequent builds and deploys will be kicked off automatically when a new commit is pushed to the GitHub repository.

## How the CI/CD scripts works

Travis CI use a single file `.travis.yml` to control both CI and CD phases.

### Continuous integration phase

Continuous integration will run when a new commit is pushed or pull request is submitted to the repository.

1. Install AZ CLI
1. Build the `Dockerfile`
   1. Expose port 80 and 2222 (SSH)
   1. Copy production files to `/var/web/`
   1. Set up SSH according to steps from this [article on docs.microsoft.com](https://docs.microsoft.com/en-us/azure/app-service/containers/tutorial-custom-docker-image#enable-ssh-connections)
   1. Set up working directory at `/var/web/`
   1. Run `npm ci`
   1. Set up entrypoint to `init.sh`
1. Test the image
   1. Run the image and host on port 80
   1. Ping http://localhost/health.txt
   1. Stop the image
1. Tag the image like `myappacr.azurecr.io/myapp-image:a1b2c3d`

### Continuous deployment phase

Continuous deployment will only start when a new commit is pushed to `master` branch.

1. Run `scripts/docker_push`
   1. Login Azure by using Service Principal
   1. Login Azure Container Registry (setup auth for Docker)
   1. Push image to Azure Container Registry
   1. Set Azure Web App to use the latest image
      - Instead of using pricey Azure Container Registry webhooks, we prefer setting up the container directly using AZ CLI

## Cleaning up

If you no longer want to run this web app, run the following steps to remove it from your Azure subscription.

### Delete the service principal

This will delete the service principal used for deployment.

```sh
az ad sp delete --id 12345678-1234-5678-abcd-12345678abcd
```

### Delete the resource group

This will delete your resource group, which contains Azure Container Registry, Azure App Service Plan for Linux, and Azure Web App for Containers.

```sh
az group delete --name myapp-rg
```

## Frequently asked questions

### Travis CI environment variables

You should have the following environment variables set in Travis CI:

| Name                          | Description                       | Secured | Sample value                                   |
|-------------------------------|-----------------------------------|---------|------------------------------------------------|
| `ACR_NAME`                    | Azure Container Registry name     | No      | `myappacr`                                    |
| `AZURE_SP_PASSWORD`           | Service principal password        | Yes     |                                                |
| `AZURE_SP_TENANT`             | Service principal tenant ID       | Yes     | GUID or `mycompany.onmicrosoft.com`            |
| `AZURE_SP_USERNAME`           | Service principal username        | Yes     | GUID or `http://azure-cli-2019-01-01-12-34-56` |
| `AZURE_WEBAPP_NAME`           | Azure Web App name                | No      | `myapp-web`                                    |
| `AZURE_WEBAPP_RESOURCE_GROUP` | Azure Web App resource group name | No      | `myapp-rg`                                     |
| `DOCKER_IMAGE_NAME`           | Docker image name                 | No      | `myapp-image`                                  |

### Diagnosing deployment issues

- For deployment log stream, https://myapp-web.scm.azurewebsites.net/api/logstream
- For SSH access into the Docker container, https://myapp-web.scm.azurewebsites.net/webssh/host

### Why enabling SSH? Is it secure?

Enabling SSH allows developers to easily diagnose issues inside the container.

Although port 2222 is exposed as SSH server, this port is secured by Azure and is not publicly accessible. If you need to access the box through SSH, you will first need to be authenticated on Azure.

[This article](https://docs.microsoft.com/en-us/azure/app-service/containers/tutorial-custom-docker-image#enable-ssh-connections) talk about SSH connections inside Docker image.

### How to rollback to previous image?

Rollback is easy. We use Git commit when versioning Docker images. You will need to find out the commit you want to rollback to:

```sh
az webapp config container set \
  --resource-group myapp-rg \
  --name myapp-web \
  --docker-custom-image-name myappacr.azurecr.io/myapp-image:a1b2c3d4e5f6
```

> Note: Git commit must be in long format.

### How about GitHub Actions?

When GitHub Actions roll out to the public, we will include GitHub Actions YAML file so you can build it on both GitHub and Travis CI.

### How can I do X?

For questions, please [submit an issue](https://github.com/compulim/azure-webapp-for-containers-template/issues) to us. We will include the answer as part of this FAQs.

## What's next?

### Deploy on Git tag

Instead of deploying on commit to `master` branch, a more professional approach would be deploy on push to Git tag.

```diff
  deploy:
    provider: script
    on:
-     branch: master
+     tags: true
    script: bash scripts/docker_push
```

Your deployment workflow will become creating a tag and pushing it.

```sh
git tag v1.0.0
git push -u origin v1.0.0
```

Optionally, you can add a GitHub Releases to automatically ZIP up your source code for archiving purpose.

You can read more about this topic from [this Travis CI article](https://docs.travis-ci.com/user/deployment/npm/#what-to-release).

## Adding a React app

For React, we will separate the build and run by using multi-stage build. This will produce a very clean production image. You can read [this article from Docker on multi-stage builds](https://docs.docker.com/develop/develop-images/multistage-build/).

> The `Dockerfile` below assumes the build script for React is `npm run build` and output is located under `/build/` folder.

```diff
+ FROM node:12 AS builder
+
+ ADD src /var/builder/src/
+ ADD public /var/builder/public/
+ ADD package*.json /var/builder/
+
+ WORKDIR /var/builder/
+ RUN \
+   npm ci \
+   && npm run build
+
  FROM node:12

  EXPOSE 80 2222

  ADD scripts/init.sh /usr/local/bin/

  # Setup OpenSSH for debugging thru Azure Web App
  # https://docs.microsoft.com/en-us/azure/app-service/containers/app-service-linux-ssh-support#ssh-support-with-custom-docker-images
  # https://docs.microsoft.com/en-us/azure/app-service/containers/tutorial-custom-docker-image
  ENV SSH_PASSWD "root:Docker!"
  ENV SSH_PORT 2222
  RUN \
    apt-get update \
    && apt-get install -y --no-install-recommends dialog \
    && apt-get update \
    && apt-get install -y --no-install-recommends openssh-server \
    && echo "$SSH_PASSWD" | chpasswd \
    && chmod u+x /usr/local/bin/init.sh

  ADD scripts/sshd_config /etc/ssh/
- ADD public /var/web/public/
+ COPY --from=builder /var/builder/build/ /var/web/

  WORKDIR /var/web/
  RUN npm install -g serve@11.1.0

  # Set up entrypoint
  ENTRYPOINT init.sh
```

## Related articles

- [Run a custom Linux container in Azure App Service]
- [Tutorial: Build a custom image and run in App Service from a private registry]

[Run a custom Linux container in Azure App Service]: https://docs.microsoft.com/en-us/azure/app-service/containers/quickstart-docker-go
[Tutorial: Build a custom image and run in App Service from a private registry]: https://docs.microsoft.com/en-us/azure/app-service/containers/tutorial-custom-docker-image
