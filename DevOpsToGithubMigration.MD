Migrating project from Azure DevOps to GitHub and setting up the deployment on Azure. Below are steps that could help ensure a smooth transition:

### Plan Refinement and Additional Steps:

1. **Create New Repo in GitHub (e.g., `myapp`)**:
   - Ensure you have the right permissions to create repositories under the GitHub account or organization.
   - Initialize the repository with a README, .gitignore (appropriate for your project's language/framework), and a license if needed.

2. **Add new remote origin to devops repo**:
   - `git remote -v`

3. **Push Existing Code from Old Repo to the New Repo Origin**:
   - `git push <neworigin> master:main`

4. **Create New Resource Group (`myapp-test`) Under Subscription-**:
   - You can do this via Azure CLI, PowerShell, or the Azure Portal.
   - Make sure your account has the necessary permissions to create resource groups and resources within the Subscription.

5. **New App Registrations in Azure for Deployment**:
   - You'll need to register a new app application "myapp-test-deployment-Account" in Azure AD to create a service principal for deployments.
      `az ad sp create --id <XXXXX-XXXXX-XXXXX>``
   - Assign the necessary roles to the service principal, like `Contributor` on the resource group where you'll be deploying.
      `az role assignment create --role contributor --subscription <my-subscription> --assignee-object-id  <object-id> --assignee-principal-type ServicePrincipal --scope /subscriptions/<my-subscription>/resourceGroups/myapp-test`
   - Create a Client secret for testing bicep deployment

6. **Create Environments in GitHub to Store Secrets**:
   - Setup GitHub environments to segregate deployment secrets for different environments (e.g. prod, test, dev).
   - Setup the environment secrets with Azure service principal credentials and any other required configuration values (e.g. AppId, SubscriptionId, TenantId).

7. **Start Creating CI and CD (Convert ARM to Bicep)**:
      Login with the deployment account using the Client secret
      `az login --service-principal -u <XXXXX-XXXXX-XXXXX> -p <my-password> --tenant <my-tenant-id>`
      1. Create ~~hostingPlanName~~ and App-Insignt with Bicep
         `az deployment group create --resource-group myapp-dev --subscription  <my-subscription> --template-file "./core/application-insights.bicep" --name "bicep-appinsights"`
      2. Deploy Key Vault Bicep
         a. Create Key Vault
         `az deployment group create --resource-group myapp-dev --subscription  <my-subscription> --template-file "./core/key-vault.bicep" --name "bicep-keyvault-$(date '+%Y%m%d%H%M%S')"`
         b. Copy the secret using the AzPowerShell module.
         `.\copySecrets.ps1 -sourceKvName "MingZAppVaultTest" -destKvName "kv-mingz-test"`
      3. Create Storage Account
         `az deployment group create --resource-group myapp-dev --subscription  <my-subscription> --template-file "./core/storage-account.bicep" --name "storage-account-$(date '+%Y%m%d%H%M%S')"`
      4. Assign Contributor role to the Resource Relay under the <shared-resources-group>
         `az role assignment create --assignee-object-id  <my-subscription> --role "Contributor" --scope "/subscriptions/<my-subscription>/resourceGroups/<shared-resources-group>/providers/Microsoft.Relay/namespaces/sb-hc-domain"`
      5. Create Function App with Bicep and deploy resources with bicep
         `az deployment group create --resource-group myapp-test --subscription  <my-subscription> --template-file "./main-stack/func-apps/template-myapp.bicep" --parameters "./main-stack/func-apps/params/bicep-myapp-params-test.json" --name "bicep-myapp-$(Get-Date -Format 'yyyyMMddHHmmss')"`
      6. CI&CD function app code to Azure using github actions.
         a. Download the publish profile from the Func App created at Step 5.
         b. Uploaded the publish profile to the github secrets.
         c. Run the github yml file to deploy build code to function app.
      7. Create ARM to bicep
         -- Covernt ARM to Bicep using the following cli
         `az bicep decompile --file <path_to_template.json>`

      8. Create each Small components
         a. Create Action Group
         `az deployment group create --resource-group myapp-dev --subscription  <my-subscription> --template-file "./core/action-group.bicep"  --name "bicep-actiongroup-$(date '+%Y%m%d%H%M%S')"`
         b. Create Alert Rules
         `az deployment group create --resource-group myapp-dev --subscription  <my-subscription> --template-file "./core/scheduled-query-rules.bicep"  --name "bicep-scheduledqueryrules-$(date '+%Y%m%d%H%M%S')"`
      10. Create CDN profiles
         a. Assign contributor role to app-registration
         `az role assignment create --role contributor --subscription <my-subscription> --assignee-object-id  <object-id> --assignee-principal-type ServicePrincipal --scope /subscriptions/<my-subscription>/resourceGroups/<shared-resources-group>`
         b. Deploy bicep
         `az deployment group create --resource-group myapp-dev --subscription  <my-subscription> --template-file "./core/cdn-profiles.bicep" --parameters "./core/params/cdn-profiles-params.json" --name "bicep-cdnprofiles"`
         c. add cname to the cdn profile

      11. Upgrade App Service Plan(Enhancment)
         a. ```az deployment group create --resource-group myapp-dev --subscription  <my-subscription> --template-file "./core/app-service-plan.bicep" --name "bicep-appserviceplan-$(date '+%Y%m%d%H%M%S')"```

      12. Rest app services
         `az role assignment create --role contributor --subscription <my-subscription> --assignee-object-id  <object-id> --assignee-principal-type ServicePrincipal --scope /subscriptions/<my-subscription>/resourceGroups/finance-integration-dev`
         `az deployment group create --resource-group myapp-dev --subscription  <my-subscription> --template-file "./core/app-service.bicep" --parameters "./core/params/app-service-params.json" --parameters environmentName="dev" --name "bicep-appservice-$(date '+%Y%m%d%H%M%S')"`
      13. Change all naming convention to use the same naming convention.
      14. All CDN and CDNE are created by one cdn-profiles.bicep
      15. Create fun-app core(Base templeate for creating func-app)
      16. Upgrade func app to use fun-app core(step#15)
      `az deployment group create --resource-group myapp-dev --subscription  <my-subscription> --template-file "./main-stack/func-myapp.bicep" --parameters "./main-stack/params/func-myapp-params.json" --name "bicep-plannertasks-$(date '+%Y%m%d%H%M%S')"`

      17. Test the main.bicep to depploy all resources at once.
      `az deployment group create --resource-group myapp-dev --subscription  <my-subscription> --template-file "./main.bicep" --name "main-bicep-$(date '+%Y%m%d%H%M%S')"  --parameters environmentName='dev'`

      18. Upgrade storage account script with DeploymentScript. This allows blob container enable static website througt bicep now.
         a. Create managed identity for the deployment script and run deployment script with managed identity.
            `az deployment group create --resource-group myapp-dev --subscription  <my-subscription> --template-file "./core/storage-account-setstaticweb.bicep" --name "bicep-setstaticweb-$(date '+%Y%m%d%H%M%S')"`
         b. if run deployment script fail, the "storage contributor" role is required and must be added to managed identity. Ask Admin to do it.
            `az role assignment create --assignee <Client-ID> --role "Storage Account Contributor" --scope /subscriptions/<my-subscription>/resourceGroups/myapp-dev`
      19. Create github action to deploy bicep script.(call main.bicep to create & update all az resources at once)
      20. Create github action to build and deploy function app code
         a. Create base yml
         b. Create function app yml
         c. Download publish profile from function app
         d. Upload publish profile to github environment secrets
         e. Run github action to deploy function app code
      21. Create github action to build and upload connect ReactJs.
         a. Get storage account key from storage account
         b. Paste the storage account key to github environment secrets
         c. yml file to build and upload code to storage account
      22. Create github action to build and upload NetCore.
         a. Get publish profile from web app
         b. Paste the publish profile to github environment secrets
         c. yml file to build and upload code to web app
         d. swap staging and production slot (action take care of this)
      23. PR validation and send notification (use action or power automation to send notification to teams)
      24. CI frontend coverage 
      25. CI backend coverage (if app size over 1GB, use large runner to handle the task)
      26. CI frontend build and deployment
      27. CI backend build and deployment
      28. Function app build and deployment

### Additional Considerations:
- **Documentation**: Update any documentation that references the old Azure DevOps setup to now point to the new processes and tooling.
- **Communication**: Inform your team about the migration and any new workflows or practices they need to follow.
