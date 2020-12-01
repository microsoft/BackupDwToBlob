# Backup (Azure Synapse) Data Warehouse to (Azure) BLOB (Storage)

## Introduction

The inspiration for this was to create an Azure Data Factory set of pipelines to backup an Azure Synapse Provisioned Pool database to Azure BLOB storage in such a way that the database will look the same when restored and return the same result from the same query over the database, or as closely as is possible.

The additional idea is to create scripts to recreate the Azure SQL Provisioned Pool database to about the sames state as it was before, or as close as is possible.

> **Design Principles**
>
> 1. Security was always to be to the highest standard, no information should be able to be gleamed by looking at the pipeline or from the results of the completed pipeline.
> 2. All code to be SQL native, no LogicApps, PowerShell or Azure Functions to be used. This was done to stick to the principle that a DBA should be able to modify and troubleshoot the process.
>

The following are targeted to be backed up (if not excluded by a schema):

* User Objects:
  * User Schemas
  * User Tables (HASH / Round Robin / Replicated)
  * User Views (This includes Materialized Views)
  * User Stored Procedures
  * User Defined Functions
  * Partitions
  * Partition Functions
  * Statistics
  * Indexes
* System Objects:
  * Statistics
  * Indexes

>Modifications can be made so that some of these parameters are not needed in the pipeline, e.g. passing the user name as a string (or SecureString) to the pipeline. As opposed retrieving it from an Azure Key Vault. This approach is obviously less secure, but Azure Key Vault could then be skipped in entirety (if so required).
>
>At this point in time I am not of the view that detailed steps for this is required and I will only use these notes to describe this less secure process. Additionally this approach does not fit into the design principles stated in the beginning.

---

## Pre-Steps

A user cannot be added to a database using the REST APIs, and therefore a user must be pre-staged in the database as per [Create a contained user mapped to Azure AD identities](https://docs.microsoft.com/en-us/azure/azure-sql/database/authentication-aad-configure?tabs=azure-powershell#create-contained-users-mapped-to-azure-ad-identities)

I have opted to use SQL Authentication in this flow. The ideal solution is to stage the ADF MSI in as an Administrator for the Azure Synapse Provisioned Pool database, but for now the flow is not setup for this.

``` SQL
-- Creates the login MyBackupUser with password 'VerySecuredPassword!123'.  
CREATE LOGIN MyBackupUser
    WITH PASSWORD = 'VerySecuredPassword!123';  
GO  

-- Creates a database user for the login created above.  
CREATE USER MyBackupUser FOR LOGIN MyBackupUser;  
GO  
```

Take the user name and password and add it to your Azure Key Vault, use the _BackupServerUserName_ and _BackupServerUserPassword_ secrets for this.

## Setup

Here I will describe how to setup the Azure Data Factory as well as all of the surrounding services.

### Azure Data Factory

The main component to be used for the backup process.

### SQL Control Database

Used to save the backup information primarily, I would surmise that this control database could also be used to stage the backup process itself (In the preliminary version I will not go into details to do that).

I used a serverless SQL Azure database with very low CPU and memory for my Control database as this is very cost effective, unless you plan on using it for more than 4 to 5 hours a day. (As always your miles may vary)

The following table is required in the control database to track the backups :

``` SQL
CREATE TABLE [dbo].[backupControl](
    [backupProcessId] [BIGINT] IDENTITY(1,1) NOT NULL,
    [dataFactoryName] [NVARCHAR](256) NOT NULL,
    [pipelineName] [NVARCHAR](256) NOT NULL,
    [pipelineRunId] [NVARCHAR](36) NOT NULL,
    [backupPath] [NVARCHAR](20) NULL,
    [rowCountsNotMatching] [INT] NOT NULL,
    [completed] [BIT] NOT NULL,
    [startTime] [DATETIME2](7) NOT NULL,
    [endTime] [DATETIME2](7) NOT NULL,
    [backupId] [INT] NOT NULL
) ON [PRIMARY]
GO
```

Use this table to track the backup location for the data warehouse.

### Azure Key Vault

Azure Key Vault (AKV) is used to secure sensitive information for the complete backup process, and maybe I did go a little overboard with adding control database and server name to the Key Vault, but once you have a pattern established you follow it to the end!

1. Add the MSI for ADF to your Key Vault as a Managed Applications Reader.
2. Add the required secrets
   * BackupBlobStorageLocation
   * BackupDatabaseScopedCredentials
   * BackupDatabaseScopedCredentialsSecret
   * BackupServerMasterKey
   * BackupServerUserName
   * BackupServerUserPassword
   * SqlControlDatabase
   * SqlControlPassword
   * SqlControlServerName
   * SqlControlUserName

---

## Azure Data Factory Pipelines

### Main Pipeline

---

![Main Pipeline](https://ibimages.blob.core.windows.net/public/BackupToBlob/MainPipeline.png)

This pipeline is used as the master pipeline to execute all of the subsequent steps, from beginning to end. It simply has 5 objects, all 5 being Execute Pipeline.

---

### Pipeline: Step 1

![Step 1](https://ibimages.blob.core.windows.net/public/BackupToBlob/Step1.png)

> Obviously you can create the Azure SQL Server beforehand and completely skip this step.

This pipeline is used to check if the backup SQL Azure Server exist, and if not, it will create it. Additionally it will add the Azure Data Factory MSI as the domain admin for this SQL Server (Only if this pipeline creates the server) and allow all Azure services to be able to connect to it.

> If the server already exist, then none of the below values would be applied.

#### Step 1 Parameters

> Almost all parameters are required and don't have any default values

| Parameter Name | Type | More Info |
| --- | --- | --- |
| infra_BackupResourceGroup | SecureString | The already existing resource group that should be used to host the new Azure SQL Server  |
| infra_BackupSqlServerName | SecureString | A valid name for the new Azure SQL Server.  |
| infra_Region | SecureString | The region where the new Azure SQL Server must be created  |
| infra_SubscriptionId | SecureString | The ID of the subscription hosting the original Azure Synapse Provisioned Pool database and the new Azure SQL Server  |
| infra_Tags | SecureString | The tags that should be added to the new Azure SQL Server, in valid JSON format. A **blank** value is acceptable  |
| security_AdfAppId | SecureString | The app ID for this Azure Data Factory  |
| security_TempAdminUserName | SecureString | A valid temporary user name for the SQL admin user |
| security_TempAdminUserPassword | SecureString | A valid password for the temporary SQL admin user |
| security_TenantId | SecureString | The tenant ID that hosts the above subscription |

1. The server cannot be created without temporary user name and password, these values can be retrieved from Key Vault.
2. The infra_Tags must be a valid JSON string in the following format: "tagKey1": "tag-value-1", "tagKey2": "tag-value-2". This value should be left blank to skip adding tags to the server

---

### Pipeline: Step 2 - Restore database to backup server

![Step 2](https://ibimages.blob.core.windows.net/public/BackupToBlob/Step2.png)

This pipeline will restore the latest restore point from the specified server and database to the specified backup server.

> Right now it is not supported to restore into another subscription in Azure

#### Step 2 Parameters

> All parameters are required and don't have any default values

| Parameter Name | Type | More Info |
| --- | --- | --- |
| infra_BackupSqlServerName | SecureString | The name of the Azure SQL Server to use to restore the backup. |
| infra_BackupResourceGroup | SecureString |   |
| infra_OriginalResourceGroup | SecureString | The name of the resource group that contains the Azure SQL Server which hosts the Azure Synapse Provisioned Pool database to be backed up   |
| infra_OriginalSqlServerName | SecureString | The URI of the Azure SQL Server that hosts the Azure Synapse Provisioned Pool database to be backed up |
| infra_Region | SecureString | The name that the restored database will have.  |
| infra_SubscriptionId | SecureString | The ID of the subscription where the Azure Synapse database is hosted  |
| synapse_BackupSku | SecureString | The name that the restored database will have.  |
| synapse_DatabaseToBeBackedUp | SecureString | The name of the database that should be restored  |
| synapse_DatabaseBackupName | SecureString | The name that the restored database will have.  |


1. Valid values for **synapse_BackupSku** is DW100c, DW200c, DW300c, DW500c, DW1000c, DW1500c and so forth, verify the sizes on the Microsoft Documentation page. (Note that the size will have a direct impact on how long a backup takes and the cost of it)
2. Additionally this only covers Gen2.
3. **infra_BackupSqlServerName** should only be the name of the server, not the complete URI.

> Region could have been taken from the original SQL Server, but it has to match up with the created server as well as the original server. (Region was added here to allow the pipelines to be absolutely atomic from each other)

> If you _just_ need a normal backup, you could simply run up to this step and pause the data warehouse at this point.

> To get a list of regions, use:
>
>``` PowerShell
>az account list-locations -o table
>```

---

### Pipeline: Step 3

![Step 3](https://ibimages.blob.core.windows.net/public/BackupToBlob/Step3.png)

This pipeline is used to prepare the restored database for back up.

The following will be done to the **Azure SQL Server**:

1. Create a Master Key, if it doesn't exist already.

``` SQL
IF (SELECT COUNT(*) FROM sys.symmetric_keys WHERE [name] LIKE '%DatabaseMasterKey%') < 1
BEGIN
    CREATE MASTER KEY ENCRYPTION BY PASSWORD = '@{activity('Get password from AKV').output.value}';
END
SELECT 1;
```

The following will be added to the **Azure Synapse Provisioned Pool database**:

1. Create 2 backup schemas, the first one is user defined (or not)
2. Set Database Scoped Credentials (DSC)
3. Create the external data source from the BLOB storage path defined
4. Create external file format (The default file format created is parquet, but you can change this step to be any valid file format. Parquet will be compressed, but is slower than CSV for export and import.)

``` SQL
--Create parquet File Format
CREATE EXTERNAL FILE FORMAT Backup_PARQUET_FileFormat  
WITH (  
    FORMAT_TYPE = PARQUET  
      , DATA_COMPRESSION = 'org.apache.hadoop.io.compress.SnappyCodec'  
    );
SELECT 1;
```

> **Kind of note to self...**
> Would be easier to change the external file name to Backup_FileFormat to make it easier to change the file type.

#### Step 3 Parameters

> All parameters are required and don't have any default values

| Parameter Name | Type | More Info |
| --- | --- | --- |
| infra_BackupSqlServer | SecureString | --- |
| synapse_DatabaseBackupName | SecureString | --- |
| sql_BackupSchemaName | SecureString | There you have it, the only parameter with a default value, which is  '\_\_\_ibBackup\_\_\_'. |
| security_AKVMasterKeyUri | SecureString | --- |
| security_AKVDatabaseScopedCredentialsUri | SecureString | --- |
| security_AKVDatabaseScopedCredentialsSecretUri | SecureString | --- |
| security_AKVBackupUserNameUri | SecureString | --- |
| security_AKVBackupUserPasswordUri | SecureString | --- |
| security_AKVBackupStorageLocationUri | SecureString | --- |
| sql_SkipSchemas | String | **Not implemented!** A comma delimited valid list of schemas that should be skipped in the backup process. |

> **Kind of note to self...**
> Will probably have to add a parameter with which Schemas can be skipped, e.g. 'Staging', 'Temp' or something like that... Never mind I added the parameter, but they scripts has to be yet modified.
> **Note to Laurence**
> The part below is me slaughtering my way through the English language...

Make sure that should you change the default backup schema, don't use a schema that is in use by the database itself as no object in this schema will be backed up. And, yes, I hear that right now you might not have any object in there that should be backed up, but we all know that some changes happen and objects are moved or added in the future. Don't be that guy!

---

### Pipeline: Step 4 - Capture database info and backup

![Step 4](https://ibimages.blob.core.windows.net/public/BackupToBlob/Step4.png)

This is the core of the whole process, this pipeline is used to do the actual backup of the data warehouse and create the scripts to restore the data warehouse. It contains a lot of T-SQL scripts for this and particular attention was paid to make sure that the structures was kept the same in the scripts as was found in the database, this includes the order of the columns as well other structures (This will be expanded to make clear which structures).

The short description is that it connects to the database and get all of the tables that should be backed up, a row count is taken and saved, this is used at the end of the backup process to ensure that all data was backed up successfully. Additionally all of the other objects are prepared to be backed up. At the end of the backup the information for this backup, most importantly the folder location for the backup, is saved to the control database. My suggestion is that the user  used for this operation simply have db_Writer permissions to uphold security principles.

#### Step 4 Parameters

> All parameters are required and don't have any default values

| Parameter Name | Type | More Info |
| --- | --- | --- |
| infra_BackupSqlServer | SecureString | The fully qualified name of the SQL Server that hosts the backup copy of the Azure Synapse database  |
| security_AKVBackupUserNameUri | SecureString | The URI to the Azure Key Vault that contains the value of the user name for the Azure Synapse back up database user account |
| security_AKVBackupUserNameUri | SecureString | The URI to the Azure Key Vault that contains the value of the user name for the Azure Synapse back up database user account |
| security_AKVBackupUserPasswordUri | SecureString | The URI to the Azure Key Vault that contains the value of the password for the Azure Synapse back up database user account |
| security_AKVControlServerNameUri | SecureString | The URI to the Azure Key Vault that contains the value for the SQL Azure Server URI that hosts the Control database |
| security_AKVControlDatabaseUri | SecureString | The URI to the Azure Key Vault that contains the value for the Control database |
| security_AKVControlPasswordUri | SecureString | The URI to the Azure Key Vault that contains the value for the password for the SQL Control database user account |
| security_AKVControlUsernameUri | SecureString | The URI to the Azure Key Vault that contains the value for the user name for the SQL Control database user account |
| sql_BackupSchemaName | SecureString | The value for the backup schema name |

---

### Pipeline: Step 5 - Delete Backup Database

![Step 4](https://ibimages.blob.core.windows.net/public/BackupToBlob/Step5.png)

## Contributing

This project welcomes contributions and suggestions. Most contributions require you to agree to a Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us the rights to use your contribution. For details, visit https://cla.opensource.microsoft.com.

When you submit a pull request, a CLA bot will automatically determine whether you need to provide a CLA and decorate the PR appropriately (e.g., status check, comment). Simply follow the instructions provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the Microsoft Open Source Code of Conduct. For more information see the Code of Conduct FAQ or contact opencode@microsoft.com with any additional questions or comments.

## Code of Conduct

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information, see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.
