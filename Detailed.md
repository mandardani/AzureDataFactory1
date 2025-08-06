## 🚀 Detailed steps
------------- Database set up & data import----------------
SQL Databases - 
		Two databases: Hospital1 & Hospital10
		both tables will have 5 tables each
		Table Names: Providers, Department, Patients, encounters & Transactions.
		
connect both tables using vscode.
use bcp command to import csv files into both database tables.
command used example:
cmd /c 'bcp [dbo].[transactions] in "C:\<local-path>\transactions.csv" -S <server> -d <database> -U h <user> -P <password> -c -t,'

use key vault to store database password
-----------------------------------------------------------
metadata driven ADF:
create load_config.csv file to store which database tables to be imported.
file will store server, database,table, is_active, load type - full/ incremental, watermark column, target folder
----------------------------------------------------------- 
Create Azure Storage Account - Hierarchical Namespace is enabled.
ADLS Gen2 - Containers 
			config - stores configuration file content list of tables to be imported 
			bronze - imported tables will be stored as parquet files
-----------------------------------------------------------
Define table to record ADF jobrun log (load_logs):

Create Azure Databricks account - this will be used further for processing data in Medallion Architecture
Databricks runtime: 14.3 LTS to support unity catalog (> 12), single node of Standard F4 is enough configuration
					Terminate after 10 minutes of inactivity

Go to Catalog: create catalog name: pipelines (External Location: config folder)
schema: jobrun, table name: load_logs

goto azure home => 
Create: Azure Databrics access connector, keep Resource ID copied
Go to Storage =>
Iam - Assign role - Azure Blob data contributor - select Azure Databricks access connector created above and assign
Go to Databricks =>
unity Catalog - create credentials - access connector id (use created above)
			  - create external location (abfss://config@storageAccount.dfs.core.windows.net)
			    use credential created above, grant permission all

all data recorded in load_logs will be stored in config folder
-------------------------------------------------------------

Create Data Factory: Launch Azure Data Factory Studio

Create Linked Services:
1. For Azure SQL Database: 
	Generic link to server - database - we will get values when pipeline reads data from config
	Database: @{linkedService().db_name}
	<user name> and password: from key vault
2. For Azure Data Lake Storage Gen2
	Go to Storage - settings - Endpoints - Data Lake Storage URL
				  -  Security + networking - Access Key
3. For Azure Databricks Delta Lake
	Go to databricks - Copy URL - this is Domain
	Launch workspace - compute and copy cluster ID from URL clusters and immediate ? after it
	clusters/0805-211117-geg50x42?o=
	To get Access token: click on profile - settings - Developer - Access Token and generate token
------------------------------------------------------------	
Create Datasets:
1. For Azure SQL Database: Generic - we will get values when pipeline reads data from config
	create parameters: db_name, schema_name, table_name
	select linked service,
	db_name	: @dataset().db_name
	Table: @dataset().schema_name.@dataset().table_name
2. For flat file: to read config file
	create parameters container, filePath, fileName
	Select linked service: Azure Data Lake Storage
	File Path: @dataset().container/ @dataset().filePath/ @dataset().fileName
3. Parquet file: to write parquet file
	create parameters container, filePath, fileName
	Select linked service: Azure Data Lake Storage
	File Path: @dataset().container/ @dataset().filePath/ @dataset().fileName
4. Azure Databricks Delta Lake: to access catalog.schema.table name
	create parameters schema_name, table_name
	Database: @dataset().schema_name, table: @dataset().table_name
-------------------------------------------------------

Create Pipeline

Activity 1: Lookup Activity (lookup_emr_config)
			source: generic flat file, read load_config.csv
Activity 2: ForEach sequential activity 
			input to this activity is output of step 1- @activity('lookup_emr_config').output.value
ForEach activity steps
2.1 generate file address using config file:
	location = bronze/@item().targetpath/@split(item().tablename,'.')[1]
2.2 Check if file is already on location
		true => 2.2.1 move file to archive folder.
				archive folder location: table_name/archive/YYYY/mm/dd
				@concat(item().targetpath,'/archive/',formatDateTime(utcNow(), 'yyyy'),'/',formatDateTime(utcNow(),'%M'),'/',formatDateTime(utcNow(),'%d'))
		false=> move to next step
2.3 Find if table is listed as Incremental or Full load
	@equals(item().loadtype, 'Full') 
	true => follow 2.3.1 - Full Load
			in case of full load, create parquet file at bronze folder, and record pipeline jobrun info
	false=> follow 2.3.2 - Incremental Load
			in case of incremental load, find out watermark column and last run time, using this info extract data from table and create parquet file and record pipeline jobrun info


