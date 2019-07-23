# Azure Web App for Containers deployed thru Travis CI

This repository is a template for setting up a simple web server on Azure Web App and deployed thru Travis CI.

We assume you have hands-on experience on several key technologies:
- Azure
   - AZ CLI
   - Azure Container Registry
   - Azure Web Apps
   - Service principal
- Docker
- GitHub
- Travis CI

## Steps

1. Azure
   1. Create resource group
      - `az group create --location "West US" --name webapptemplate`
   1. Create Azure Container Registry
      - `az acr create --name webapptemplate --resource-group webapptemplate --sku Basic --admin-enabled --location "West US"`
   1. Create Service Principal
      - `az ad sp create-for-rbac --name http://webapptemplate`
   1. Create Azure Web App for Containers
      - https://ms.portal.azure.com/#create/Microsoft.AppSvcLinux
   1. Configure Service Principal in Azure Web App
      - Select "Access Control (IAM)"
      - Click "+ Add" and then "Add role assignment"
      - In "Role", select "Contributor"
      - In "Select", type "webapptemplate"
      - Click "Save"
1. Travis CI
   1. Configure Travis CI and set environment variables
      - Refer to the section [Travis CI environment variables](#travis-ci-environment-variables)

## Diagnostics

You will be able to SSH into your Azure Web App at https://webapptemplate.scm.azurewebsites.net/webssh/host.

## Travis CI environment variables

| Name | Description | Secured |
| - | - | - |
| `ACR_NAME` | Azure Container Registry name | No |
| `AZURE_SP_PASSWORD` | Service principal password | Yes |
| `AZURE_SP_TENANT` | Service principal tenant ID | Yes |
| `AZURE_SP_USERNAME` | Service principal username | Yes |
| `AZURE_WEBAPP_NAME` | Azure Web App name | No |
| `AZURE_WEBAPP_RESOURCE_GROUP` | Azure Web App resource group name | No |
| `CONTAINER_NAME` | Container name | No |
