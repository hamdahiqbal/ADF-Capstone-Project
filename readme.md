# ADF Publish Branch — ARM Deployment Reference
## Automated CI/CD Artifact Store for the Enterprise Data Lakehouse

---

> This branch is automatically maintained by Azure Data Factory. Every time a developer clicks **Publish All** in the ADF Studio, the contents of this branch are regenerated to reflect the latest state of the factory's control plane. Do not manually edit files in this branch. Any manual changes will be overwritten without warning on the next publish operation.

> The source of truth for all pipeline development, dataset definitions, documentation, and source files lives in the **main** branch. This branch exists exclusively for deployment.

---

## Table of Contents

1. [What This Branch Is — And Why It Exists](#what-this-branch-is--and-why-it-exists)
2. [Understanding ARM Templates](#understanding-arm-templates)
3. [Repository Structure](#repository-structure)
4. [What Gets Deployed](#what-gets-deployed)
5. [Pre-Deployment Requirements](#pre-deployment-requirements)
6. [Step-by-Step Deployment Guide](#step-by-step-deployment-guide)
7. [Parameter Reference](#parameter-reference)
8. [How the Publish Workflow Operates](#how-the-publish-workflow-operates)
9. [Operational Risk Mitigation](#operational-risk-mitigation)
10. [The Two-Branch Model — Developer Guide](#the-two-branch-model--developer-guide)
11. [Related Resources](#related-resources)

---

## What This Branch Is — And Why It Exists

When you build pipelines in Azure Data Factory, everything you create — every pipeline, dataset, linked service, data flow, and trigger — is stored as a JSON definition file in the connected Git repository. While you are developing, these files live in the **main** branch and go through a normal review and approval process just like any other code.

However, not everything in the main branch is ready to be deployed. A developer might have half-finished work, a broken pipeline mid-refactor, or experimental changes that have not been tested yet. The **main** branch is the workshop; the **adf_publish** branch is the shipping dock.

When a data engineer is satisfied that the factory's current state is stable and correct, they click **Publish All** in the ADF Studio. ADF reads every resource definition from the main branch, validates them, compiles them into a single ARM (Azure Resource Manager) deployment template, and commits that template to this branch. The `adf_publish` branch therefore always contains the last known-good, validated state of the entire factory.

The practical result of this design is powerful: any Azure environment — a development sandbox, a user acceptance testing (UAT) environment, or a production system — can be instantly recreated from scratch by pointing one ARM deployment command at this branch. This is Infrastructure as Code in practice.

---

## Understanding ARM Templates

An ARM (Azure Resource Manager) Template is a JSON file that describes your entire Azure infrastructure declaratively. "Declaratively" means you describe the desired end state, not the sequence of steps to get there. You say "I want a Data Factory called hi-data-factory with these pipelines" and Azure figures out how to create it.

**Why this matters for DevOps and CI/CD:**

Without ARM templates, recreating a factory means manually clicking through the Azure Portal — creating linked services, then datasets, then pipelines, then data flows, in the right order, every time. This process takes hours, is error-prone, and is impossible to automate.

With an ARM template, the entire factory is described in a single, version-controlled JSON file. You can deploy the complete factory to a brand-new environment in under ten minutes using a single command. You can also compare two ARM templates side by side to see exactly what changed between deployments — just like comparing two versions of source code.

**The two files in every ARM template deployment:**

The `ARMTemplateForFactory.json` file contains the full infrastructure definition — every resource, every property, every expression. However, it leaves certain sensitive values (passwords, storage keys, connection strings) as empty placeholders. This is intentional: the template itself is committed to a public or shared repository, so it must not contain credentials.

The `ARMTemplateParametersForFactory.json` file is the companion that fills in those placeholders. It is a small file containing only the parameter values. In a production CI/CD pipeline, this file's values are injected at deployment time from a secret manager (Azure Key Vault, GitHub Secrets, or Azure DevOps Variable Groups). For manual deployments, you fill in the values directly.

---

## Repository Structure

```
hi-data-factory/
|
|-- ARMTemplateForFactory.json
|       The primary deployment artifact. A single self-contained ARM template
|       containing definitions for all pipelines, datasets, linked services,
|       data flows, and triggers in the factory. Use this file for standard
|       single-template deployments.
|
|-- ARMTemplateParametersForFactory.json
|       The parameter companion to the primary template. Contains all
|       configurable values that differ between environments (names, URLs)
|       and all sensitive secrets (passwords, keys) as empty strings to
|       be filled in at deployment time.
|
|-- globalParameters/
|   |-- hi-data-factory_GlobalParameters.json
|           Contains factory-level global parameters that are shared across
|           all pipelines. These are deployed separately from the main
|           ARM template and applied to the factory after initial deployment.
|
|-- linkedTemplates/
    |-- ArmTemplate_master.json
    |       The entry point for a linked-template deployment strategy.
    |       This approach splits the factory definition across multiple
    |       smaller template files to stay within ARM's 4MB single-template
    |       size limit for large factories.
    |
    |-- ArmTemplate_0.json
    |       Contains pipeline, dataset, and linked service definitions.
    |
    |-- ArmTemplate_1.json
    |       Contains data flow and trigger definitions.
    |
    |-- ArmTemplateParameters_master.json
            The parameter file for the linked-template deployment.
```

---

## Main Branch Structure (Development Source)

The main branch is the development home for this project. Every folder listed below is committed and maintained by the engineering team. When Publish All is triggered, ADF reads from this branch to generate the ARM templates stored here.

```
ADF-Capstone-Project-main/
|
|-- pipeline/
|       Contains one JSON file per pipeline. Each file defines the full activity
|       graph, dependencies, and parameter schema for that pipeline.
|       |-- api_injection.json
|       |-- load_to_sql.json
|       |-- pl_gold_layer.json
|       |-- pl_master_orchestration.json
|       |-- pl_onprem_to_bronze.json
|       |-- pl_silver_layer.json
|       |-- sql_to_data_lake.json
|
|-- dataflow/
|       Contains the Mapping Data Flow definitions (Spark transformation graphs).
|       |-- df_transform_silver.json
|       |-- df_analytics_gold.json
|
|-- dataset/
|       Contains one JSON file per dataset. Each file is a pointer to a specific
|       data location with its format, schema, and parameter definitions.
|
|-- linkedService/
|       Contains one JSON file per linked service (connection definition).
|       Covers ADLS Gen2, Azure SQL, HTTP (GitHub), and the on-premises file system.
|
|-- integrationRuntime/
|       Contains the Self-Hosted Integration Runtime definition (hi-shir).
|       This registers the on-premises bridge with the factory.
|
|-- trigger/
|       Contains the schedule trigger definition (tr_daily_orchestration).
|
|-- factory/
|       Contains the factory-level settings file including the Git configuration
|       that links ADF Studio to this repository.
|
|-- source_files/
|       The on-premises source data files used as inputs for the Bronze ingestion
|       pipeline. These are the raw files that the SHIR reads from the local machine.
|       |-- DimAirline.csv          (Airline reference data — loaded to airline_mart)
|       |-- DimAirport.json         (Airport reference data — Bronze landing only)
|       |-- DimFlight.csv           (Flight reference data — Bronze landing only)
|       |-- DimPassenger.csv        (Passenger data — loaded to passenger_mart and Silver)
|       |-- empty.json              (Empty placeholder used for API sink schema testing)
|       |-- fact_bookings_full.sql  (DDL + seed data script for the FactBookings SQL table)
|       |-- last-load.json          (Initial watermark state file — must be uploaded to
|                                    bronze/monitor/ in ADLS Gen2 before first pipeline run)
|
|-- documentation/
|       Twelve phase-specific markdown files covering the full implementation guide,
|       architecture diagrams, step-by-step instructions, and risk mitigation tables.
|       Files: phase1_resources.md through phase12_git_devops.md
|
|-- screenshots/
|       Organised verification screenshots for every phase of the implementation,
|       referenced directly by the documentation markdown files.
|
|-- README.md
        The master engineering guide. Covers all 12 phases end-to-end, with conceptual
        explanations, architecture diagrams, and links to every phase document.
```

---

## What Gets Deployed

The following table documents every resource captured in this ARM template. This represents the complete factory state at the time of the last Publish All operation.

### Linked Services

| Resource Name | Type | Target System | Authentication |
|:---|:---|:---|:---|
| `ls_data_lake` | Azure Data Lake Storage Gen2 | hiadfstorage (ADLS Gen2) | Account Key |
| `ls_onprem_file` | File System | Local Windows File System | Windows Credentials via SHIR |
| `ls_sql` | Azure SQL Database | hi-db (Azure SQL) | SQL Authentication |
| `ls_github` | HTTP | raw.githubusercontent.com | Anonymous |

### Integration Runtimes

| Resource Name | Type | Host |
|:---|:---|:---|
| `AutoResolveIntegrationRuntime` | Azure (Managed) | Azure Cloud |
| `hi-shir` | Self-Hosted | Local Windows Machine |

### Datasets

| Resource Name | Format | Layer | Purpose |
|:---|:---|:---|:---|
| `ds_onprem_binary` | Binary (Parameterized) | Source | On-premises file extraction template |
| `ds_bronze_binary` | Binary (Parameterized) | Bronze | Bronze layer landing target for binary files |
| `ds_monitor_lastload` | JSON | Bronze/Monitor | High-water mark state file (`last-load.json`) |
| `ds_sql_source` | Azure SQL Table | Source | `FactBookings` table for incremental extraction |
| `ds_sql_sink` | DelimitedText (ADLS) | Bronze | CSV landing path for SQL-sourced data |
| `ds_github_api` | JSON (HTTP) | Source | GitHub API raw file endpoint |
| `ds_bronze_api` | JSON (ADLS) | Bronze | Bronze landing path for API payloads |
| `ds_bronze_bookings` | DelimitedText (ADLS) | Bronze | Bookings CSV for Silver transformation |
| `ds_bronze_passengers` | DelimitedText (ADLS) | Bronze | Passenger CSV for Silver transformation |
| `ds_sql_airline_mart` | Azure SQL Table | Serving | `dbo.airline_mart` destination table |
| `ds_sql_passenger_mart` | Azure SQL Table | Serving | `dbo.passenger_mart` destination table |

### Pipelines

| Resource Name | Stage | Pattern | Core Activities |
|:---|:---|:---|:---|
| `pl_onprem_to_bronze` | Bronze | Metadata-Driven ForEach | ForEach + Copy (Binary) |
| `api_injection` | Bronze | HTTP GET to Lake | Copy (HTTP to JSON) |
| `sql_to_data_lake` | Bronze | High-Water Mark Incremental | Lookup + Lookup + Copy + Copy |
| `load_to_sql` | Serving | Parallel Relational Load | Copy (Airline) + Copy (Passenger) |
| `pl_silver_layer` | Silver | Spark Transformation | Data Flow (df_transform_silver) |
| `pl_gold_layer` | Gold | Spark Analytical Aggregation | Data Flow (df_analytics_gold) |
| `pl_master_orchestration` | Orchestration | Sequential Parent-Child | Execute Pipeline (x3) + Web (Alert) |

### Data Flows (Mapping / Apache Spark)

| Resource Name | Sources | Transformation Chain | Sink |
|:---|:---|:---|:---|
| `df_transform_silver` | BronzeBookings, BronzePassengers | Source > Source > Join (Inner, passenger_id) > DerivedColumn (upper(country)) > AlterRow (upsertIf true) | Silver Delta Lake (`silver/bookings_delta`) |
| `df_analytics_gold` | SilverBookings (Delta), BronzeAirline (CSV) | Source > Source > Join (Left Outer, airline_id) > Aggregate (sum ticket_cost) > Window (denseRank desc) > Filter (Rank <= 5) | Gold Delta Lake (`gold/business_view/top_airlines`) |

### Triggers

| Resource Name | Type | Schedule | Target Pipeline |
|:---|:---|:---|:---|
| `tr_daily_orchestration` | Schedule | Daily, Asia/Karachi timezone | `pl_master_orchestration` |

---

## Pre-Deployment Requirements

Before running the deployment, confirm the following infrastructure and permissions are in place.

**Azure Subscription and Permissions:**
- An active Azure subscription with Contributor or Owner access on the target resource group.
- The resource group must already exist. This ARM template deploys factory resources only, not the resource group itself.

**Existing Infrastructure (must already be provisioned before deploying):**
- An Azure Data Lake Storage Gen2 account with Hierarchical Namespace enabled, and three containers named `bronze`, `silver`, and `gold`.
- An Azure SQL Database with the `FactBookings` table populated. The DDL scripts for the mart tables (`airline_mart`, `passenger_mart`) must also be executed:
  ```sql
  CREATE TABLE airline_mart (
      airline_id   INT,
      airline_name VARCHAR(100)
  );
  CREATE TABLE passenger_mart (
      passenger_id INT,
      first_name   VARCHAR(100),
      last_name    VARCHAR(100),
      gender       VARCHAR(10),
      nationality  VARCHAR(50)
  );
  ```
- A Windows machine (local or Azure VM) with the Self-Hosted Integration Runtime application installed, registered, and showing a Connected status. The SHIR must be reachable from the target factory instance.
- The source CSV files (`DimAirline.csv`, `DimAirport.json`, `DimFlight.csv`, `DimPassenger.csv`) in the configured on-premises folder path.
- The `last-load.json` watermark file uploaded to the `bronze/monitor/` path in ADLS Gen2 with the following content:
  ```json
  { "last_load": "1900-01-01" }
  ```

**Local Tools (for CLI deployment):**
- Azure CLI version 2.20 or later. Verify with `az --version`.
- PowerShell 7 or later (optional, for script-based deployment).
- Git, to clone this branch.

---

## Step-by-Step Deployment Guide

### Option 1 — Azure CLI (Recommended for Automation)

**Step 1: Authenticate with Azure**
```bash
az login
az account set --subscription "<your-subscription-id>"
```

**Step 2: Clone only this branch**
```bash
git clone --branch adf_publish --single-branch ^
  https://github.com/hamdahiqbal/ADF-Capstone-Project.git
cd ADF-Capstone-Project/hi-data-factory
```

**Step 3: Fill in the parameter file**

Open `ARMTemplateParametersForFactory.json` and replace the empty string values for all three secret parameters:
```json
{
    "ls_data_lake_accountKey": "<your-adls-storage-account-key>",
    "ls_onprem_file_password": "<your-windows-account-password>",
    "ls_sql_password":        "<your-azure-sql-password>"
}
```

**Step 4: Deploy to your resource group**
```bash
az deployment group create \
  --resource-group <your-resource-group-name> \
  --template-file ARMTemplateForFactory.json \
  --parameters ARMTemplateParametersForFactory.json
```

**Step 5: Apply Global Parameters (if applicable)**

If the factory uses global parameters, apply them after deployment:
```bash
az datafactory update \
  --resource-group <your-resource-group-name> \
  --factory-name hi-data-factory \
  --global-parameters @globalParameters/hi-data-factory_GlobalParameters.json
```

**Step 6: Verify the deployment**
1. Navigate to the Azure Portal and open the newly deployed Data Factory.
2. Open ADF Studio and confirm all 7 pipelines, 11 datasets, and 2 data flows are visible.
3. Open each Linked Service and click **Test Connection** to confirm all four connections are valid.
4. Run the master orchestration pipeline in Debug mode to confirm end-to-end dataflow.

---

### Option 2 — Azure Portal (Manual, No CLI Required)

1. Navigate to the Azure Portal: `portal.azure.com`
2. Search for **Deploy a custom template** in the search bar.
3. Click **Build your own template in the editor**.
4. Click **Load file** and upload `ARMTemplateForFactory.json`.
5. Click **Save** to return to the deployment form.
6. Click **Edit parameters** and upload `ARMTemplateParametersForFactory.json`. Fill in the three secret values.
7. Select the correct Subscription and Resource Group.
8. Click **Review + create**, then **Create**.

---

## Parameter Reference

The complete list of parameters defined in `ARMTemplateParametersForFactory.json`.

| Parameter Name | Type | Required | Description |
|:---|:---|:---|:---|
| `factoryName` | String | Yes | The name of the Azure Data Factory instance. Default: `hi-data-factory`. |
| `ls_data_lake_accountKey` | SecureString | Yes | Primary access key for the Azure Data Lake Storage Gen2 account. |
| `ls_onprem_file_host` | String | Yes | UNC or local path to the on-premises source folder. Example: `C:\Users\YourName\source_files` |
| `ls_onprem_file_userId` | String | Yes | The Windows username for the SHIR host machine. |
| `ls_onprem_file_password` | SecureString | Yes | The Windows account password for the SHIR host machine. |
| `ls_sql_server` | String | Yes | Fully qualified Azure SQL server hostname. Example: `hi-server.database.windows.net` |
| `ls_sql_database` | String | Yes | Name of the Azure SQL Database. Example: `hi-db` |
| `ls_sql_userName` | String | Yes | Azure SQL login username. |
| `ls_sql_password` | SecureString | Yes | Azure SQL login password. |
| `ls_github_url` | String | Yes | Base URL for the GitHub raw API linked service. Default: `https://raw.githubusercontent.com/` |
| `trigger_start_time` | String | Yes | ISO 8601 UTC start time for the daily schedule trigger. Example: `2025-01-01T03:30:00Z` |

---

## How the Publish Workflow Operates

It is important to understand what happens behind the scenes when a developer clicks Publish All in ADF Studio. This knowledge is essential for maintaining the repository correctly.

**Step 1 — Development happens in main:**
Developers work on the main branch. When they add a new pipeline, ADF creates a new JSON file in the `pipeline/` folder of the main branch via a standard Git commit. Pull requests are used to review and merge changes, just like any other software project.

**Step 2 — Publish All is triggered:**
When the team is ready to release a stable build, any authorised user clicks Publish All in ADF Studio. This action is equivalent to a "release to production" event.

**Step 3 — ADF validates all resources:**
Before generating any templates, ADF validates every resource definition in the main branch. If there are missing references (for example, a pipeline references a dataset that does not exist), the publish operation fails with a validation error. Nothing is deployed. This validation gate prevents broken configurations from reaching the deployment branch.

**Step 4 — ARM templates are generated and committed:**
If validation passes, ADF compiles all resource definitions into the ARM template format and commits the resulting files directly to the `adf_publish` branch. This commit is made by the ADF service principal automatically — you will see it in the Git history as a commit from the ADF service, not from any individual developer.

**Step 5 — The adf_publish branch is ready for deployment:**
The newly committed ARM templates in this branch now represent a validated, deployable snapshot of the entire factory. Any CI/CD system (Azure DevOps, GitHub Actions) can be configured to watch for new commits on this branch and automatically trigger a deployment to the target environment.

---

## Operational Risk Mitigation

| Criticality | Risk | Mitigation |
|:---:|:---|:---|
| **CRITICAL** | Committing secrets directly into `ARMTemplateParametersForFactory.json` | All three secret parameters (`accountKey`, `password` x2) must be provided at deployment time only. Never commit a parameter file containing real credentials to this or any public repository. Use Azure Key Vault references or CI/CD secret injection. |
| **CRITICAL** | Manually editing files in `adf_publish` | All manual edits will be overwritten and permanently lost on the next Publish All operation. All development must happen in the main branch. |
| **HIGH** | Deploying to a resource group without the prerequisite infrastructure | The ARM template deploys the factory resources only. The ADLS Gen2 account, SQL Database, and SHIR must exist before deployment. Deploying without them will cause all Linked Services to fail their connection tests. |
| **HIGH** | SHIR machine is offline during deployment | The factory will deploy successfully, but the `ls_onprem_file` Linked Service will show a yellow warning indicating the IR is unreachable. The `pl_onprem_to_bronze` pipeline will fail until the SHIR machine is back online and the runtime shows Connected. |
| **MODERATE** | Trigger fires immediately after deployment | The `tr_daily_orchestration` trigger is deployed in a Stopped state by default and must be manually activated after deployment. Verify the trigger is not in a Started state before activating it in a new environment. |
| **MODERATE** | Wrong region selected for the new deployment | All resources (ADF, ADLS Gen2, Azure SQL) must be in the same Azure region to avoid cross-region data transfer costs and latency. |

---

## The Two-Branch Model — Developer Guide

This project uses a standard two-branch model enforced by ADF's Git integration. Understanding the rules governing each branch is essential for all contributors.

```
main branch                             adf_publish branch
--------------------                    ------------------------
Purpose: Development                    Purpose: Deployment
Owner: All developers                   Owner: Azure Data Factory (automated)
Write access: Via Pull Request          Write access: ADF service only
Contains: Source JSON definitions       Contains: Compiled ARM templates
Updated by: Developer commits           Updated by: Publish All action
Reviewed by: Peer code review           Reviewed by: No manual review (automated)
```

**The golden rule:** Never commit directly to `adf_publish`. If you need to change what gets deployed, make the change in main, get it reviewed, merge it, and then Publish All. The publish action will update `adf_publish` automatically.

**The review checkpoint:** Because the `adf_publish` branch is the deployment gate, the Publish All action should be a deliberate, team-approved decision — not something done casually by any individual. In mature teams, Publish All is protected behind a role permission and requires a second sign-off.

---

## Related Resources

| Resource | Description |
|:---|:---|
| [main branch](https://github.com/hamdahiqbal/ADF-Capstone-Project/tree/main) | Source code — all pipeline JSON files, datasets, documentation |
| [Main README](https://github.com/hamdahiqbal/ADF-Capstone-Project/blob/main/README.md) | Comprehensive 12-phase engineering implementation guide |
| [documentation/](https://github.com/hamdahiqbal/ADF-Capstone-Project/tree/main/documentation) | Phase-by-phase technical documentation with architecture diagrams |
| [Azure ADF Git Integration Docs](https://learn.microsoft.com/en-us/azure/data-factory/source-control) | Official Microsoft documentation on ADF source control and publish workflow |
| [ARM Template Deployment Docs](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/deploy-cli) | Official Azure CLI deployment reference |
