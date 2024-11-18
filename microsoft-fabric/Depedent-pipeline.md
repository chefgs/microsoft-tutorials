## Creating Dependent Fabric pipeline

It is possible in Azure DevOps to have a pipeline that is dependent on another pipeline and can be triggered after the dependent pipeline is completed. This can be achieved using pipeline triggers.

### Steps to Break Down the Consolidated Pipeline into Independent Pipelines

1. **Create Independent Pipelines**: Create separate YAML files for each independent pipeline.
2. **Set Up Pipeline Triggers**: Use pipeline triggers to trigger the dependent pipelines after the completion of the initial pipeline.

### Example Breakdown

#### Pipeline 1: Build and Deploy Warehouse

```yaml
# File: .azure-pipelines/build-and-deploy-warehouse.yml
trigger:
  branches:
    include:
      - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  resourceGroupName: 'your-resource-group'
  location: 'your-location'
  sqlServerName: 'your-sql-server-name'
  sqlAdminUser: 'your-sql-admin-user'
  sqlAdminPassword: '$(sqlAdminPassword)' # Securely stored in Azure DevOps pipeline secrets

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
              azureSubscription: 'your-azure-subscription'
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

#### Pipeline 2: Deploy Semantic Model

```yaml
# File: .azure-pipelines/deploy-semantic-model.yml
resources:
  pipelines:
    - pipeline: buildAndDeployWarehouse
      source: build-and-deploy-warehouse
      trigger: true

pool:
  vmImage: 'ubuntu-latest'

variables:
  azureServiceConnection: 'your-azure-service-connection'
  resourceGroupName: 'your-resource-group'
  location: 'your-location'
  semanticModelName: 'your-semantic-model-name'

jobs:
  - job: DeploySemanticModel
    steps:
      - checkout: self
      - task: AzureCLI@2
        inputs:
          azureSubscription: $(azureServiceConnection)
          scriptType: 'bash'
          scriptLocation: 'inlineScript'
          inlineScript: |
            # Deploy Semantic Model
            az synapse workspace create --name $(semanticModelName) --resource-group $(resourceGroupName) --location $(location)
```

#### Pipeline 3: Deploy Notebook

```yaml
# File: .azure-pipelines/deploy-notebook.yml
resources:
  pipelines:
    - pipeline: deploySemanticModel
      source: deploy-semantic-model
      trigger: true

pool:
  vmImage: 'ubuntu-latest'

variables:
  azureServiceConnection: 'your-azure-service-connection'
  databricksServiceEndpoint: 'your-databricks-service-endpoint'
  databricksToken: $(PAT_TOKEN)

jobs:
  - job: DeployNotebook
    steps:
      - checkout: self
      - task: DatabricksNotebook@0
        inputs:
          azureSubscription: $(azureServiceConnection)
          databricksServiceEndpoint: $(databricksServiceEndpoint)
          authenticationToken: $(databricksToken)
          notebookPath: '/Workspace/Notebooks/your-notebook'
          notebookLanguage: 'PYTHON'
          notebookContent: '$(System.DefaultWorkingDirectory)/path/to/your/notebook.py'
```

### Additional Pipelines

You can follow the same pattern to create additional independent pipelines for deploying Lakehouse, SQL, Data Pipelines, Dataflow Gen2, Reports, Datasets, Dashboards, Workspaces, Data Sources, Security Configurations, and Deployment Pipelines. Each pipeline will have a `resources` section to specify the dependent pipeline and a `trigger: true` to enable automatic triggering.

### Example for Deploying Lakehouse

```yaml
# File: .azure-pipelines/deploy-lakehouse.yml
resources:
  pipelines:
    - pipeline: deployNotebook
      source: deploy-notebook
      trigger: true

pool:
  vmImage: 'ubuntu-latest'

variables:
  azureServiceConnection: 'your-azure-service-connection'
  resourceGroupName: 'your-resource-group'
  location: 'your-location'
  lakehouseName: 'your-lakehouse-name'

jobs:
  - job: DeployLakehouse
    steps:
      - checkout: self
      - task: AzureCLI@2
        inputs:
          azureSubscription: $(azureServiceConnection)
          scriptType: 'bash'
          scriptLocation: 'inlineScript'
          inlineScript: |
            # Create Lakehouse
            az synapse lakehouse create --name $(lakehouseName) --resource-group $(resourceGroupName) --location $(location)
```

By breaking down the consolidated pipeline into independent pipelines and using pipeline triggers, you can create a more modular and maintainable CI/CD process in Azure DevOps.

---

## Fabric Component Dependency Model

### Microsoft Fabric Work Items Dependency Hierarchy

1. **DeployWarehouse**
   - **Dependencies**: None
   - **Reason**: The SQL Server and SQL Database (Warehouse) are foundational components that other services can depend on for data storage and retrieval.

2. **DeploySemanticModel**
   - **Dependencies**: DeployWarehouse
   - **Reason**: The semantic model may rely on the data stored in the SQL Warehouse for creating data models and analytics.

3. **DeployNotebook**
   - **Dependencies**: DeploySemanticModel
   - **Reason**: Notebooks often perform data processing and analytics that may depend on the semantic model and the underlying data in the warehouse.

4. **DeployLakehouse**
   - **Dependencies**: DeployNotebook
   - **Reason**: The Lakehouse integrates data from various sources, including processed data from notebooks.

5. **DeploySQL**
   - **Dependencies**: DeployLakehouse
   - **Reason**: The SQL endpoint may be used to query data stored in the Lakehouse.

6. **DeployDataPipeline**
   - **Dependencies**: DeploySQL
   - **Reason**: Data pipelines in Azure Data Factory may depend on SQL endpoints for data extraction, transformation, and loading (ETL) processes.

7. **DeployDataflowGen2**
   - **Dependencies**: DeployDataPipeline
   - **Reason**: Dataflows in Power BI often depend on data pipelines to prepare and transform data before it is loaded into Power BI.

8. **DeployReport**
   - **Dependencies**: DeployDataflowGen2
   - **Reason**: Power BI reports rely on dataflows to provide the necessary data for visualizations and analytics.

9. **DeployDataset**
   - **Dependencies**: DeployReport
   - **Reason**: Datasets in Power BI are often created based on the data provided by reports and dataflows.

10. **DeployDashboard**
    - **Dependencies**: DeployDataset
    - **Reason**: Dashboards aggregate visualizations from multiple reports and datasets to provide a high-level overview.

11. **DeployWorkspace**
    - **Dependencies**: None (can be created independently)
    - **Reason**: A Power BI workspace is a container for reports, datasets, and dashboards. It can be created independently but is necessary for deploying other Power BI artifacts.

12. **DeployDataSource**
    - **Dependencies**: DeployWorkspace
    - **Reason**: Data sources are often created within a workspace to provide connectivity to various data services.

13. **ConfigureSecurity**
    - **Dependencies**: DeployWorkspace, DeployDataSource
    - **Reason**: Security configurations are applied to workspaces and data sources to control access and permissions.

14. **DeployDeploymentPipeline**
    - **Dependencies**: ConfigureSecurity
    - **Reason**: Deployment pipelines are used to manage the deployment of various artifacts and require security configurations to be in place to ensure proper access control.

### Explanation of Dependencies

- **Foundational Components**: Components like SQL Server, SQL Database (Warehouse), and Power BI Workspace are foundational and need to be set up first as they provide the necessary infrastructure for other components.
- **Data Flow**: The flow of data from ingestion (Data Pipelines) to transformation (Notebooks, Dataflows) to visualization (Reports, Dashboards) creates a natural dependency chain.
- **Security and Access Control**: Security configurations are applied after the creation of workspaces and data sources to ensure that access controls are properly enforced.
- **Deployment Management**: Deployment pipelines are configured last to manage the deployment of all other components in a controlled and secure manner.

By understanding these dependencies, you can structure your CI/CD pipelines in Azure DevOps to ensure that each component is deployed in the correct order, ensuring a smooth and efficient deployment process.

---

### Text Diagram of Dependencies

```plaintext
DeployWarehouse
  |
  v
DeploySemanticModel
  |
  v
DeployNotebook
  |
  v
DeployLakehouse
  |
  v
DeploySQL
  |
  v
DeployDataPipeline
  |
  v
DeployDataflowGen2
  |
  v
DeployReport
  |
  v
DeployDataset
  |
  v
DeployDashboard
```

### Table Format of Dependencies

| Work Item                | Dependencies                  | Reason                                                                 |
|--------------------------|-------------------------------|-----------------------------------------------------------------------|
| **DeployWarehouse**      | None                          | Foundational component for data storage.                              |
| **DeploySemanticModel**  | DeployWarehouse               | Relies on data stored in the warehouse.                               |
| **DeployNotebook**       | DeploySemanticModel           | Performs data processing and analytics.                               |
| **DeployLakehouse**      | DeployNotebook                | Integrates data from various sources.                                 |
| **DeploySQL**            | DeployLakehouse               | SQL endpoint for querying data.                                       |
| **DeployDataPipeline**   | DeploySQL                     | ETL processes depend on SQL endpoints.                                |
| **DeployDataflowGen2**   | DeployDataPipeline            | Dataflows depend on data pipelines for data preparation.              |
| **DeployReport**         | DeployDataflowGen2            | Reports rely on dataflows for visualizations.                         |
| **DeployDataset**        | DeployReport                  | Datasets are created based on report data.                            |
| **DeployDashboard**      | DeployDataset                 | Dashboards aggregate visualizations from reports and datasets.        |

---

## Common for Workspace Items

```
DeployWorkspace
  |
  v
DeployDataSource
  |
  v
ConfigureSecurity
  |
  v
DeployDeploymentPipeline
```


| Work Item                | Dependencies                  | Reason                                                                 |
|--------------------------|-------------------------------|-----------------------------------------------------------------------|
| **DeployWorkspace**      | None                          | Container for reports, datasets, and dashboards.                      |
| **DeployDataSource**     | DeployWorkspace               | Created within a workspace for connectivity to data services.         |
| **ConfigureSecurity**    | DeployWorkspace, DeployDataSource | Applies security configurations to workspaces and data sources.       |
| **DeployDeploymentPipeline** | ConfigureSecurity         | Manages the deployment of all other components with proper access control. |

By using this text diagram and table format, you can clearly visualize and understand the dependencies between different Microsoft Fabric work items, ensuring a smooth and efficient deployment process in Azure DevOps.