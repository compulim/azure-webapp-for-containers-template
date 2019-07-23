# Azure Web App for Containers app

This repository is a template for setting up a simple web server on Azure Web App using Docker single image CI/CD from GitHub and Travis CI.

We assume you have hands-on experience on these key technologies:

- Azure
   - AZ CLI
   - Azure Container Registry
   - Azure Web Apps
   - Service principal
- Docker
- GitHub
- Travis CI

You can extend this template to use Docker Compose or Kubernetes on Azure Web App.

The simple web server used in this template is [`serve`](https://www.npmjs.com/package/serve) from NPM. You can modify `Dockerfile` to change to another language or hosting server.

This template enables SSH access to the Docker image via Azure Web App (a.k.a. SCM or Kudu). It is a secure service provided by Azure.

## How it works

- GitHub repository keep your project files, including `Dockerfile`, related scripts and runtime files
   - In `Dockerfile`, notably
      - Copy and build to `/var/web/`
      - Expose port 80 (web) and 2222 (SSH for Kudu)
      - Set up SSH for Kudu
      - Run `init.sh`
   - In `init.sh`, notably
      - Start SSH server on port 2222
      - Start the web server in `/var/web/` by calling `npm start`
- On every push, GitHub will trigger Travis CI
- Travis CI have two phases: build and deploy
   1. In build phase, we will build the Docker image and tag it `myapp-acr.azurecr.io/myapp:a1b2c3d`
      - The tag is based on Azure Container Registry name and Git commit SHA
   1. In deployment phase, we will deploy to Azure using the `docker_push` script
      1. Log into Azure and Azure Container Registry
      1. Push the Docker image to Azure Container Registry
      1. Obtain Azure Container Registry credentials
      1. Configure the container image on Azure Web App

## Setup steps

1. Azure
   1. Create resource group to hold all your resources
      - `az group create --location "West US" --name myapp-rg`
   1. Create Azure Container Registry
      - `az acr create --name myapp-acr --resource-group myapp-rg --sku Basic --admin-enabled --location "West US"`
   1. Create Service Principal
      - `az ad sp create-for-rbac --name http://myapp-spn`
      - Remember to write down the password
   1. Create Azure Web App for Containers
      - https://ms.portal.azure.com/#create/Microsoft.AppSvcLinux
      - For container settings, please use default values (Nginx in single image), we will override them in our Travis CI deployment script
   1. Configure Service Principal in Azure Web App
      - Select "Access Control (IAM)"
      - Click "+ Add" and then "Add role assignment"
      - In "Role", select "Contributor"
      - In "Select", type "http://myapp-spn"
      - Click "Save"
1. Travis CI
   1. Configure Travis CI and set environment variables
      - Refer to the section [Travis CI environment variables](#travis-ci-environment-variables)

## Diagnostics

- Log stream at https://webapptemplate.scm.azurewebsites.net/api/logstream
- SSH access at https://webapptemplate.scm.azurewebsites.net/webssh/host
- Kudu at https://webapptemplate.scm.azurewebsites.net/

## Travis CI environment variables

In order to deploy to Azure, you will need to set up the following environment variables in your Travis CI project settings page.

| Name                          | Description                       | Secured | Sample value                |
|-------------------------------|-----------------------------------|---------|-----------------------------|
| `ACR_NAME`                    | Azure Container Registry name     | No      | `myapp-acr`                 |
| `AZURE_SP_PASSWORD`           | Service principal password        | Yes     |                             |
| `AZURE_SP_TENANT`             | Service principal tenant ID       | Yes     | `mycompany.onmicrosoft.com` |
| `AZURE_SP_USERNAME`           | Service principal username        | Yes     | `http://myapp-spn`          |
| `AZURE_WEBAPP_NAME`           | Azure Web App name                | No      | `myapp`                     |
| `AZURE_WEBAPP_RESOURCE_GROUP` | Azure Web App resource group name | No      | `myapp-rg`                  |
| `DOCKER_IMAGE_NAME`           | Docker image name                 | No      | `myapp`                     |
