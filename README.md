# Modern Data Engineering with Medallion Architecture using DBT, Databricks, Spark and Azure Cloud
 In this project, we setup and end to end data engineering using Apache Spark, Azure Databricks, Data Build Tool (DBT) using Azure as our cloud provider. This project illustrate the process of data ingestion to the lakehouse, data integration with ADF and data transformation with Databricks, and DBT.

1. <b>Architecture Diagram</b>:
    - <p><img src="images/architecture_diagram.jpg" alt="architecture_diagram" width="800px"></p>

## <ins>Current Environment</ins>
- Cloud Provider — Azure
- Database — Azure SQL
- Azure Data Lake
- Azure Key Vault
- Azure Databricks
- Azure Data Factory
- dbt (Data Build Tool)

## <ins>Setting up the stage on Azure Cloud</ins>
- Setting up the resource groups
  <p><img src="images/18.rg_with_databricks.jpg" alt="resources" width="800px"></p>
- Setting up the Storage Accounts on ADLS Gen2
- Setting up Azure Data Factory
- Setting up Azure Key Vault
- Creating SQL Database and Loading Sample Data
  <p><img src="images/3.SQL_created.jpg" alt="3.SQL_created" width="800px"></p>

## <ins>Data Integration with Azure Data Factory</ins>
Once all the setup is done, the next thing is to start creating pipelines on ADF.

- Connecting ADF to Azure SQL Database
  <p><img src="images/5.Connection_successful.jpg" alt="5.Connection_successful" width="800px"></p>
- Connecting ADF to Azure Data Lake Gen2
  <p><img src="images/7.linked_service_adsl_successful.jpg" alt="7.linked_service_adsl_successful" width="800px"></p>
- Adding Dynamic Content to the mix:
  Dynamic Content allows us to have a parameterised placeholder in our pipeline. We’ll be creating two datasets, TablesQuery and SqlTable each of which will help us fetch data from the SQL database dynamically and with parameters.
* TablesQuery: You can create a new dataset using the AzureSqlDatabase1 linked service created earlier to fetch the list of tables from the information schema on our DB dynamically. We create this dataset without selecting a table name.
* SqlTable: We’ll also need to create the SqlTable dataset using the AzureSqlDatabase1 linked service as well. This will also be done without selecting a table name but with parameters
  <p><img src="images/12.set_parameters.jpg" alt="12.set_parameters" width="800px"></p>
  <p><img src="images/15.SqlTable_done.jpg" alt="15.SqlTable_done" width="800px"></p>
- Going back to the pipeline, we’ll need to drag the lookup activity to the pipeline plane and in there, we’ll configure the lookup to use the TablesQuery dataset. In the settings section, we’ll select the Query radio button and pass in our query:
```sql
SELECT * FROM [ertan-medallion-db-dev].information_schema.tables
WHERE TABLE_TYPE = 'BASE TABLE' and TABLE_SCHEMA = 'SalesLT';
```
  <p><img src="images/9.TablesQuery.successful.jpg" alt="9.TablesQuery.successful" width="800px"></p>

- Iterating through the Lookup Results and ForEach Activity: The foreach activity give us access to individual records in the array of tables information from the lookup. So we drag and drop ForEach activity into the pipeline we configure the items using dynamic content.
- Dumping Table Records into the Bronze Layer: Once the ForEach activity is configured to use the @activity(‘Fetch All Tables’).output.value , the next thing is to configure the operations that will be happening to each of the items in the array. So we drag in Copy Data activity from the move and transform section into the for each sub-pipeline
<p><img src="images/10..jpg" alt="10." width="800px"></p>
- This is configured to use the second dataset created (SqlTable) which allows parameters to be passed into it. Here’s the source configuration.
<p><img src="images/13.set_parameters_name.jpg" alt="13.set_parameters_name" width="800px"></p>

- For the sink, we’ll need to create a Parquet File output from our Azure Data Lake Storage Gen2 . This will be leveraging on the already established linked service previously setup. We’ll pass in bronze into the file path and leave the directory and filename empty as they will be filled with parameters.
<p><img src="images/14.1.jpg" alt="14.1" width="800px"></p>
<p><img src="images/14.set_sink_but_first_dataset.jpg" alt="14.set_sink_but_first_dataset" width="800px"></p>
- In our pipeline, we configure the sink and pass in the required parameters specified (FolderName and FileName) and the file path will be bronze.
<p><img src="images/16.1.ParquetFileOutput.jpg" alt="16.1.ParquetFileOutput" width="800px"></p>
- The only thing left to do to complete the Copy Data Activity is to select the sink of ParquetFileOutput from the dropdown. Here are the parameters passed into the FolderName and FileName respectively

```sql
@formatDateTime(utcNow(), 'yyyy-MM-dd')

@concat(item().TABLE_SCHEMA, '.', item().TABLE_NAME, '.parquet')
```
<p><img src="images/14.2.sink_done.jpg" alt="14.2.sink_done" width="800px"></p>
- If you trigger (debug) this pipeline, you will be able to see the data dumped into the bronze layer correctly.
<p><img src="images/16.2.succeeded.jpg" alt="16.2.succeeded" width="800px"></p>
<p><img src="images/16.4.loaded_in_bronze.jpg" alt="16.4.loaded_in_bronze" width="800px"></p>

- So the data was uploaded successfully into the bronze layer.

## <ins>Setting up Databricks</ins>
You will need to search for “Databricks” in the search bar or find it in the navigation pane. Ensure you select and fill in the required details and once the deployment is complete, go to the resource and click on the “Launch Workspace” button. This will open the Azure Databricks workspace in a new tab.

- To setup Databricks on Azure, search for “Databricks” in the search bar. Ensure you select and fill in the required details and once the deployment is complete, go to the resource and click on the “Launch Workspace” button.
  
- Linking Key Vault to Azure Databricks:  Key Vault holds all your secrets, to add the secrets you need to connect databricks to the DNS and Resource endpoint of Key Vault. This can be done by adding #secrets/createScope at the end of your workspace url.
  <p><img src="images/21.2.jpg" alt="21.2" width="800px"></p>
- Go to key vault and create a secret  
  <p><img src="images/20.Create_secret.jpg" alt="20.Create_secret" width="800px"></p>
- To get the DNS Name and Resource ID, you can get this in your Key Vault properties. (Key Vaults > [Key vault name] > Settings > Properties). The Vault URI is the DNS Name and Resource ID is the Resource ID
  <p><img src="images/21.1.jpg" alt="21.1" width="800px"></p>
- In order to work with Databricks, you have to check the connection  whether was successful and you have to mount the storage account directories. You can find the code under the name BaseNotebook in the Notebooks folder.
- Second notebook will be using table schema to create a database if it doesn’t exist (saleslt in our case) and the individual tables as well using the parquest dumped into the bronze layer.
- Before move back to Data Factory. We need to setup the access token that will be used to connect to our Databricks. You can get this under profile > user settings > developer > generate access token.
- Now, we can go back to our Pipeline with our access token, we can now connect our Notebook to Databricks. We need to create a new linked service for our Azure Databricks account. This can be done by click on the Azure Databricks Tab and New . Fill in the details and the token generated earlier to continue.
  <p><img src="images/22.connection_adf_databricks.jpg" alt="22.connection_adf_databricks" width="800px"></p>
- Select the Notebook (in our case, bronze to catalog db) and add the parameters that will be sent across while running the notebook. Here are the parameters table_schema, table_name, fileName and here are the values
```sql
@item().table_schema
@item().table_name
@formatDateTime(utcNow(), 'yyyy-MM-dd')
```
  <p><img src="images/22.1.parameters.jpg" alt="22.1.parameters" width="800px"></p>
- If you trigger the pipeline if there are no errors, our data should be loaded properly.
  <p><img src="images/23.databricks_success.jpg" alt="23.databricks_success" width="800px"></p>
  <p><img src="images/24_catalog_db_loaded_successfully.jpg" alt="24_catalog_db_loaded_successfully" width="800px"></p>

  

