## Passing Variables to Pipeline

To pass variables to an Azure DevOps pipeline from outside the YAML file, you can use variable groups or pipeline variables defined in Azure DevOps. This allows you to manage variables separately and keep your pipeline YAML file generic.

### Using Variable Groups

1. **Create a Variable Group**:
   - Navigate to `Pipelines` > `Library` in Azure DevOps.
   - Click `+ Variable group` to create a new variable group.
   - Add variables to the group.

2. **Link Variable Group in YAML**:
   ```yaml
   variables:
   - group: 'your-variable-group-name'
   ```

### Using Pipeline Variables

1. **Define Pipeline Variables**:
   - Navigate to your pipeline in Azure DevOps.
   - Click `Edit` and then `Variables`.
   - Add variables directly in the pipeline UI.

2. **Access Pipeline Variables in YAML**:
   ```yaml
   variables:
     resourceGroupName: $(resourceGroupName)
     location: $(location)
     sqlServerName: $(sqlServerName)
     sqlAdminUser: $(sqlAdminUser)
     sqlAdminPassword: $(sqlAdminPassword)
   ```

### Example Pipeline YAML Using Variable Groups

```yaml
# File: .azure-pipelines/build-and-deploy-warehouse.yml
trigger:
  branches:
    include:
      - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  - group: 'your-variable-group-name'

stages:
- stage: Build
  jobs:
  - job: Build
    steps:
    - task: UseDotNet@2
      inputs:
        packageType: 'sdk'
        version: '5.x'
        installationPath: $(Agent.ToolsDirectory)/dotnet

    - script: |
        dotnet build
      displayName: 'Build Project'

- stage: Deploy
  jobs:
  - deployment: DeployWarehouse
    environment: 'production'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureCLI@2
            inputs:
              azureSubscription: $(azureServiceConnection)
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                # Create SQL Server
                az sql server create --name $(sqlServerName) --resource-group $(resourceGroupName) --location $(location) --admin-user $(sqlAdminUser) --admin-password $(sqlAdminPassword)
                
                # Create SQL Database (Warehouse)
                az sql db create --resource-group $(resourceGroupName) --server $(sqlServerName) --name $(warehouseName) --service-objective S0

                # Apply schema changes (if any)
                sqlcmd -S tcp:$(sqlServerName).database.windows.net,1433 -d $(warehouseName) -U $(sqlAdminUser) -P $(sqlAdminPassword) -i schema.sql

              addSpnToEnvironment: true
```

### Explanation

1. **Variable Group**: The `variables` section includes a reference to a variable group named `your-variable-group-name`.
2. **Pipeline Variables**: Variables like `resourceGroupName`, `location`, `sqlServerName`, `sqlAdminUser`, and `sqlAdminPassword` are accessed using the `$(variableName)` syntax.

### Steps to Implement

1. **Create Variable Group**:
   - Go to `Pipelines` > `Library` in Azure DevOps.
   - Create a new variable group and add the necessary variables.

2. **Link Variable Group in YAML**:
   - Add the variable group reference in the `variables` section of your pipeline YAML file.

3. **Define Pipeline Variables**:
   - Alternatively, define variables directly in the pipeline UI if you prefer not to use variable groups.

By using variable groups or pipeline variables, you can manage your variables separately from the pipeline YAML file, making your pipeline more generic and easier to maintain.