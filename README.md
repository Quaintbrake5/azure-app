# Deploy a Node.js Express App to Azure App Service with GitHub Actions

## Overview

In this lab, you will deploy a simple Node.js/Express application to **Azure App Service** using:

- GitHub
- GitHub Actions
- Azure App Service
- Azure Managed Identity
- OpenID Connect (OIDC)
- Continuous Deployment (CI/CD)

The final workflow will look like this:

```
GitHub Repository
       │
       │ git push
       ▼
GitHub Actions
       │
       │ OIDC authentication
       ▼
Azure Managed Identity
       │
       │ Deployment permission
       ▼
Azure App Service
       │
       ▼
Running Express Application
```

> This guide assumes Azure App Service has already been created successfully and that you did NOT encounter the "Cannot find SourceControlToken with name GitHub" error during setup.

---

## 1. Prerequisites

You should have:

- An Azure account
- An Azure App Service
- A GitHub account
- A GitHub repository
- Node.js installed locally
- Azure CLI installed or access to Azure Cloud Shell
- Git installed

You should also know:

- Your Azure Resource Group name
- Your Azure App Service name
- Your GitHub repository
- Your GitHub branch name

Example:

```text
Resource Group:
cloudcomputinglab

App Service:
azure-web-app2

GitHub Repository:
YOUR_USERNAME/YOUR_REPOSITORY

Branch:
main
```

## 2. Create a Simple Express Application

Create a project folder:

```bash
mkdir azure-app
cd azure-app
```

Initialize npm:

```bash
npm init -y
```

Install Express:

```bash
npm install express
```

## 3. Create the Express Server

Create:

```
server.js
```

Add:

```javascript
const express = require("express");

const app = express();

const PORT = process.env.PORT || 3000;

app.get("/", (req, res) => {
    res.send(`
        <h1>Hello from Azure App Service!</h1>
        <p>This application is running on Azure.</p>
    `);
});

app.get("/students", (req, res) => {
    res.json([
        {
            name: "Student 1",
            course: "Python"
        },
        {
            name: "Student 2",
            course: "Java"
        }
    ]);
});

app.listen(PORT, () => {
    console.log(`Server running on port ${PORT}`);
});
```

## 4. Configure package.json

Your `package.json` should contain a start script.

Example:

```json
{
  "name": "azure-app",
  "version": "1.0.0",
  "description": "Simple Express application deployed to Azure",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^5.1.0"
  }
}
```

The important part is:

```json
"scripts": {
    "start": "node server.js"
}
```

Azure App Service uses this to start your application.

## 5. Test the Application Locally

Run:

```bash
npm start
```

You should see:

```
Server running on port 3000
```

Open:

```
http://localhost:3000
```

You should see:

```
Hello from Azure App Service!
```

Also test:

```
http://localhost:3000/students
```

You should receive JSON similar to:

```json
[
  {
    "name": "Student 1",
    "course": "Python"
  },
  {
    "name": "Student 2",
    "course": "Java"
  }
]
```

## 6. Create a GitHub Repository

Create a repository on GitHub.

For example:

```
azure-app
```

Do not upload secrets such as:

- Passwords
- API keys
- Azure credentials
- Private keys
- Connection strings containing passwords

## 7. Push the Project to GitHub

Initialize Git:

```bash
git init
```

Add the files:

```bash
git add .
```

Commit:

```bash
git commit -m "initial Express application"
```

Connect your GitHub repository:

```bash
git remote add origin https://github.com/YOUR_USERNAME/YOUR_REPOSITORY.git
```

Push:

```bash
git branch -M main
git push -u origin main
```

Your repository should now contain something similar to:

```
azure-app/
│
├── server.js
├── package.json
├── package-lock.json
└── .gitignore
```

## 8. Connect GitHub to Azure App Service

Open the Azure Portal.

Go to:

> Azure Portal → App Services → YOUR APP SERVICE → Deployment Center

Choose:

- Source: GitHub

Choose:

- Build provider: GitHub Actions

Select:

- GitHub account
- Repository
- Branch: main

For the runtime:

- Node

Use a current supported Node version.

For example:

- Node 24

Azure will generate a GitHub Actions workflow.

## 9. Understand the Generated Workflow

Azure may generate a file similar to:

```
.github/
└── workflows/
    └── main_azure-web-app.yml
```

The workflow generally has two jobs:

```
Build
  ↓
Deploy
```

The Build job:

- Checks out the repository
- Installs Node.js
- Installs dependencies
- Creates a deployment artifact

The Deploy job:

- Downloads the artifact
- Authenticates with Azure
- Deploys the application to App Service

## 10. Recommended GitHub Actions Workflow

Your workflow can look like this:

```yaml
name: Build and deploy Node.js app to Azure Web App

on:
  push:
    branches:
      - main

  workflow_dispatch:

jobs:

  build:
    runs-on: ubuntu-latest

    permissions:
      contents: read

    steps:

      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "24.x"

      - name: Install dependencies
        run: npm install

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: node-app
          path: .

  deploy:
    runs-on: ubuntu-latest

    needs: build

    permissions:
      id-token: write
      contents: read

    steps:

      - name: Download artifact
        uses: actions/download-artifact@v4
        with:
          name: node-app

      - name: Login to Azure
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy to Azure Web App
        uses: azure/webapps-deploy@v3
        with:
          app-name: YOUR_APP_SERVICE_NAME
          slot-name: Production
          package: .
```

Replace:

```
YOUR_APP_SERVICE_NAME
```

with your actual Azure App Service name.

## 11. Understanding the Azure Credentials

Do NOT put Azure credentials directly inside your YAML file.

Do NOT do this:

```yaml
client-id: "actual-client-id"
tenant-id: "actual-tenant-id"
subscription-id: "actual-subscription-id"
```

Instead, use GitHub Secrets:

```yaml
client-id: ${{ secrets.AZURE_CLIENT_ID }}
tenant-id: ${{ secrets.AZURE_TENANT_ID }}
subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

This keeps the credentials out of your source code.

## 12. Create the GitHub Secrets

Go to your GitHub repository:

> Settings → Secrets and variables → Actions → New repository secret

Create:

- `AZURE_CLIENT_ID`
- `AZURE_TENANT_ID`
- `AZURE_SUBSCRIPTION_ID`

The values come from Azure.

You can retrieve your subscription ID with:

```bash
az account show --query id -o tsv
```

Retrieve your tenant ID:

```bash
az account show --query tenantId -o tsv
```

If using a user-assigned managed identity, retrieve its client ID:

```bash
az identity show \
  --name YOUR_IDENTITY_NAME \
  --resource-group YOUR_RESOURCE_GROUP \
  --query clientId \
  -o tsv
```

Never publish these values in your GitHub repository.

## 13. GitHub Actions OIDC Authentication

The workflow uses:

```yaml
permissions:
  id-token: write
  contents: read
```

The important permission is:

```yaml
id-token: write
```

This allows GitHub Actions to request an OIDC token.

The authentication flow is:

```
GitHub Actions
      │
      │ OIDC token
      ▼
Microsoft Entra ID
      │
      │ Federated Identity
      ▼
User-Assigned Managed Identity
      │
      │ Azure RBAC permissions
      ▼
Azure App Service
```

This means you do not need to put an Azure password or service principal secret into GitHub.

## 14. Create a User-Assigned Managed Identity

If Azure did not automatically create the required identity, you can create one with Azure CLI.

First make sure you are logged into Azure:

```bash
az login
```

Check your subscription:

```bash
az account show
```

Create the identity:

```bash
az identity create \
  --name YOUR_IDENTITY_NAME \
  --resource-group YOUR_RESOURCE_GROUP
```

Example:

```bash
az identity create \
  --name azure-app-github \
  --resource-group cloudcomputinglab
```

## 15. Give the Identity Permission to Deploy

The managed identity needs permission to deploy to the App Service.

Retrieve the principal ID:

```bash
az identity show \
  --name YOUR_IDENTITY_NAME \
  --resource-group YOUR_RESOURCE_GROUP \
  --query principalId \
  -o tsv
```

Then assign the Website Contributor role:

```bash
az role assignment create \
  --assignee-object-id "$(az identity show \
    --name YOUR_IDENTITY_NAME \
    --resource-group YOUR_RESOURCE_GROUP \
    --query principalId -o tsv)" \
  --assignee-principal-type ServicePrincipal \
  --role "Website Contributor" \
  --scope "/subscriptions/YOUR_SUBSCRIPTION_ID/resourceGroups/YOUR_RESOURCE_GROUP/providers/Microsoft.Web/sites/YOUR_APP_SERVICE_NAME"
```

Replace the placeholders with your own values.

## 16. Create the Federated Identity Credential

This is one of the most important parts of the setup.

The federated identity credential tells Azure:

> "Trust GitHub Actions from this specific repository and branch."

The issuer is:

```
https://token.actions.githubusercontent.com
```

The audience is:

```
api://AzureADTokenExchange
```

The subject must match the subject GitHub sends.

For many repositories, this looks like:

```
repo:USERNAME/REPOSITORY:ref:refs/heads/main
```

However, newer GitHub repositories may use an immutable repository identity in the OIDC subject.

For example, GitHub Actions may display something like:

```
repo:USERNAME@REPOSITORY_ID/REPOSITORY@OWNER_ID:ref:refs/heads/main
```

Do not guess the subject.

Look at the GitHub Actions log:

```
Federated token details:
subject claim - ...
```

Copy the subject exactly and use that value when configuring the federated credential.

Example:

```bash
az identity federated-credential create \
  --name github-app \
  --identity-name YOUR_IDENTITY_NAME \
  --resource-group YOUR_RESOURCE_GROUP \
  --issuer "https://token.actions.githubusercontent.com" \
  --subject "YOUR_EXACT_GITHUB_SUBJECT" \
  --audiences "api://AzureADTokenExchange"
```

## 17. Verify the Federated Credential

Run:

```bash
az identity federated-credential list \
  --identity-name YOUR_IDENTITY_NAME \
  --resource-group YOUR_RESOURCE_GROUP \
  -o json
```

Check:

- issuer
- subject
- audiences

The important values are:

```
Issuer:
https://token.actions.githubusercontent.com

Audience:
api://AzureADTokenExchange
```

The subject must match the GitHub Actions OIDC subject exactly.

## 18. Important: The Client ID Must Match the Identity

A very common error is using the client ID from an old managed identity.

For example:

```
Old identity
     ↓
old-client-id
```

and:

```
New identity
     ↓
new-client-id
```

If the federated credential belongs to the new identity, GitHub must authenticate using the new identity's client ID.

Check it with:

```bash
az identity show \
  --name YOUR_IDENTITY_NAME \
  --resource-group YOUR_RESOURCE_GROUP \
  --query clientId \
  -o tsv
```

Make sure that value is stored in:

```
AZURE_CLIENT_ID
```

on GitHub.

## 19. Trigger the Deployment

Once everything is configured, push to main:

```bash
git add .
git commit -m "deploy application to Azure"
git push origin main
```

GitHub Actions should automatically start.

You can view it:

> GitHub → Repository → Actions → Your workflow

You should see:

```
Build
  ✓ Checkout
  ✓ Setup Node
  ✓ npm install
  ✓ Upload artifact

Deploy
  ✓ Download artifact
  ✓ Azure login
  ✓ Deploy to Azure
```

## 20. Manually Run the Workflow

Because the workflow contains:

```yaml
workflow_dispatch:
```

you can also run it manually.

Go to:

> GitHub → Actions → Your workflow → Run workflow

Select:

- main

Then click:

- Run workflow

## 21. View Deployment Logs in Azure

After GitHub Actions finishes successfully:

Go to:

> Azure Portal → App Services → YOUR APP SERVICE → Deployment Center

You can see deployment information there.

You can also use:

> Monitoring → Log stream

to view the application's startup logs.

## 22. Understanding the Azure Log Stream

A successful Node.js App Service startup may show:

```
APP SERVICE ON LINUX
```

followed by:

```
NodeJS Version : v24.x.x
```

Then Azure starts your application.

If your startup command is:

```
npm start
```

you should eventually see your application start.

For example:

```
Server running on port 8080
```

## 23. Important: PORT

Your Express application should NOT hard-code the Azure port.

Use:

```javascript
const PORT = process.env.PORT || 3000;
```

This is correct.

Locally:

```
PORT = 3000
```

On Azure:

```
PORT = Azure-provided port
```

So this is recommended:

```javascript
const PORT = process.env.PORT || 3000;

app.listen(PORT, () => {
    console.log(`Server running on port ${PORT}`);
});
```

## 24. Configure the Startup Command

In Azure:

> App Service → Configuration → General settings → Startup Command

For this application, use:

```
npm start
```

Your `package.json` must contain:

```json
"scripts": {
    "start": "node server.js"
}
```

## 25. Common Error: Cannot Find index.js

You may see:

```
Error: Cannot find module '/home/site/wwwroot/index.js'
```

This means Azure is trying to start:

```
index.js
```

but your project may actually contain:

```
server.js
```

Fix your startup command/package.json.

For example:

```json
"scripts": {
    "start": "node server.js"
}
```

Then configure Azure to run:

```
npm start
```

## 26. Common Error: npm test Failed

You may see:

```
npm install

added packages...

> azure-app@1.0.0 test
> echo "Error: no test specified" && exit 1

Error: no test specified
Error: Process completed with exit code 1.
```

This happens if your GitHub Actions workflow contains:

```yaml
- run: npm test
```

but your `package.json` does not contain a test.

For a beginner Express application, you can either remove the test step or add a real test.

For example, remove:

```yaml
- name: Run tests
  run: npm test
```

if testing is not part of the lab yet.

## 27. Common Error: AADSTS700213

You may see:

```
AADSTS700213:
No matching federated identity record found
```

This means Azure could not find a federated identity credential matching the GitHub OIDC token.

Check these three things:

1. Issuer
   ```
   https://token.actions.githubusercontent.com
   ```
2. Audience
   ```
   api://AzureADTokenExchange
   ```
3. Subject

The Azure federated credential subject must match the GitHub Actions subject exactly.

Look at:

> GitHub Actions → Failed workflow → azure/login

Find:

```
Federated token details:
subject claim - ...
```

Compare that value with:

```bash
az identity federated-credential list \
  --identity-name YOUR_IDENTITY_NAME \
  --resource-group YOUR_RESOURCE_GROUP \
  -o json
```

## 28. Common Error: Wrong Client ID

If you created multiple managed identities, make sure GitHub is using the correct one.

For example:

```
azure-app-github
```

might have:

```
Client ID A
```

while:

```
azure-old-app-github
```

has:

```
Client ID B
```

The GitHub secret:

```
AZURE_CLIENT_ID
```

must contain the client ID belonging to the identity that contains your federated credential.

## 29. Common Error: Node 20 Deprecation Warning

GitHub may display:

```
Node 20 is being deprecated.
This workflow is running with Node 24 by default.
```

This is generally a warning about the Node runtime used internally by a GitHub Action.

It is separate from your application's Node version.

Your application can still use:

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: "24.x"
```

Do NOT confuse this warning with an Azure authentication error.

## 30. Common Error: SourceControlToken

You may encounter:

```
Failed to set up deployment:
Cannot find SourceControlToken with name GitHub
```

This is a different problem from the OIDC deployment process described in this guide.

If you encounter this during Azure's Deployment Center setup, stop and resolve the GitHub/Azure source-control connection first.

Once the App Service deployment configuration is successfully created, the workflow can be managed directly through:

```
.github/workflows/
```

in your GitHub repository.

## 31. Check the Final Application

Once deployment succeeds, Azure gives your App Service a URL similar to:

```
https://YOUR_APP_SERVICE_NAME.azurewebsites.net
```

Open:

```
https://YOUR_APP_SERVICE_NAME.azurewebsites.net
```

You should see:

```
Hello from Azure App Service!
```

Test the API:

```
https://YOUR_APP_SERVICE_NAME.azurewebsites.net/students
```

You should receive:

```json
[
  {
    "name": "Student 1",
    "course": "Python"
  },
  {
    "name": "Student 2",
    "course": "Java"
  }
]
```

## 32. Test Continuous Deployment

Make a change to your Express application.

For example, change:

```html
<h1>Hello from Azure App Service!</h1>
```

to:

```html
<h1>Hello from our Cloud Computing Lab!</h1>
```

Commit:

```bash
git add .
git commit -m "update application message"
git push origin main
```

GitHub Actions should automatically run again.

The process is:

```
git push
    ↓
GitHub Actions starts
    ↓
Build
    ↓
Install dependencies
    ↓
Create artifact
    ↓
Authenticate with Azure
    ↓
Deploy
    ↓
Azure App Service restarts
    ↓
Updated application is live
```

## 33. Final Project Structure

A basic project should look like:

```
azure-app/
│
├── .github/
│   └── workflows/
│       └── main_azure-web-app.yml
│
├── node_modules/
│
├── package-lock.json
├── package.json
├── server.js
└── .gitignore
```

You should NOT commit:

```
node_modules/
```

Add it to `.gitignore`:

```
node_modules
.env
```

## 34. Final Checklist

Before saying the deployment is complete, verify:

**Application**

- [ ] Express installed
- [ ] server.js exists
- [ ] package.json exists
- [ ] npm start works locally
- [ ] process.env.PORT is used
- [ ] / route works
- [ ] /students route works

**GitHub**

- [ ] Repository created
- [ ] Code pushed to main
- [ ] .github/workflows/ contains the workflow
- [ ] Workflow has id-token: write
- [ ] Workflow uses azure/login@v2
- [ ] Workflow uses azure/webapps-deploy@v3

**GitHub Secrets**

- [ ] AZURE_CLIENT_ID
- [ ] AZURE_TENANT_ID
- [ ] AZURE_SUBSCRIPTION_ID

Never commit these values to Git.

**Azure**

- [ ] App Service exists
- [ ] Correct App Service name is in YAML
- [ ] Managed identity exists
- [ ] Managed identity has deployment permissions
- [ ] Federated credential exists
- [ ] Issuer is correct
- [ ] Audience is correct
- [ ] Subject exactly matches GitHub's OIDC subject
- [ ] Startup command is correct

**Deployment**

- [ ] GitHub Actions Build succeeds
- [ ] Azure login succeeds
- [ ] Deployment succeeds
- [ ] Azure Log Stream shows application startup
- [ ] App Service URL works
- [ ] /students endpoint works
- [ ] Pushing a new commit triggers another deployment

## 35. What You Have Built

You have now created a basic CI/CD pipeline:

```
             DEVELOPER
                 │
                 │ git push
                 ▼
          ┌───────────────┐
          │    GitHub     │
          │   Repository  │
          └───────┬───────┘
                  │
                  ▼
          ┌───────────────┐
          │ GitHub Actions│
          │               │
          │  Build        │
          │  npm install  │
          │  Artifact     │
          └───────┬───────┘
                  │
                  │ OIDC
                  ▼
          ┌───────────────┐
          │ Azure Managed │
          │   Identity    │
          └───────┬───────┘
                  │
                  │ RBAC
                  ▼
          ┌───────────────┐
          │ Azure App     │
          │   Service     │
          └───────┬───────┘
                  │
                  ▼
          ┌───────────────┐
          │ Express App   │
          │               │
          │ /             │
          │ /students     │
          └───────────────┘
```

This is a practical example of:

- Cloud computing
- Platform as a Service (PaaS)
- Continuous Integration (CI)
- Continuous Deployment (CD)
- GitHub Actions
- Azure App Service
- Managed Identity
- OpenID Connect
- Role-Based Access Control (RBAC)
- Node.js
- Express.js

The major advantage is that after the initial setup, developers can simply:

```bash
git add .
git commit -m "update application"
git push
```

and GitHub Actions automatically builds and deploys the new version to Azure.