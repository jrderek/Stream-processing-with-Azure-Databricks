# Stream-processing-with-Azure-Databricks
This reference architecture shows an end-to-end stream processing pipeline. This type of pipeline has four stages: ingest, process, store, and analysis and reporting. For this reference architecture, the pipeline ingests data from two sources, performs a join on related records from each stream, enriches the result, and calculates an average in real time. The results are stored for further analysis.



Scenario: A taxi company collects data about each taxi trip. For this scenario, we assume there are two separate devices sending data. The taxi has a meter that sends information about each ride — the duration, distance, and pickup and dropoff locations. A separate device accepts payments from customers and sends data about fares. To spot ridership trends, the taxi company wants to calculate the average tip per mile driven, in real time, for each neighborhood.

Deploy the solution

A deployment for this reference architecture is available on GitHub.

Prerequisites
Clone, fork, or download this GitHub repository.

Install Docker to run the data generator.

Install Azure CLI 2.0.

Install Databricks CLI.

From a command prompt, bash prompt, or PowerShell prompt, sign into your Azure account as follows:

az login
Optional - Install a Java IDE, with the following resources:

JDK 1.8
Scala SDK 2.12
Maven 3.6.3
Note: Instructions are included for building via a docker container if you do not want to install a Java IDE.

Download the New York City taxi and neighborhood data files
Create a directory named DataFile in the root of the cloned Github repository in your local file system.

Open a web browser and navigate to https://uofi.app.box.com/v/NYCtaxidata/folder/2332219935.

Click the Download button on this page to download a zip file of all the taxi data for that year.

Extract the zip file to the DataFile directory.

Note: This zip file contains other zip files. Don't extract the child zip files.

The directory structure should look like the following:

/DataFile
    /FOIL2013
        trip_data_1.zip
        trip_data_2.zip
        trip_data_3.zip
        ...
Open a web browser and navigate to https://www.census.gov/geographies/mapping-files/time-series/geo/cartographic-boundary.html#ti1400387013.

Under the section County Subdivisions click the dropdown an select New York.

Copy the cb_2019_36_cousub_500k.zip file from your browser's downloads directory to the DataFile directory.

Deploy the Azure resources
From a shell or Windows Command Prompt, run the following command and follow the sign-in prompt:

az login
Navigate to the folder named azure in the GitHub repository directory:

cd azure
Run the following commands to deploy the Azure resources:

export resourceGroup='[Resource group name]'
export resourceLocation='[Region]'
export eventHubNamespace='[Event Hubs namespace name]'
export databricksWorkspaceName='[Azure Databricks workspace name]'
export cosmosDatabaseAccount='[Cosmos DB database name]'
export logAnalyticsWorkspaceName='[Log Analytics workspace name]'
export logAnalyticsWorkspaceRegion='[Log Analytics region]'

# Create a resource group
az group create --name $resourceGroup --location $resourceLocation

# Deploy resources
az deployment group create --resource-group $resourceGroup \
 --template-file ./deployresources.json --parameters \
 eventHubNamespace=$eventHubNamespace \
    databricksWorkspaceName=$databricksWorkspaceName \
 cosmosDatabaseAccount=$cosmosDatabaseAccount \
 logAnalyticsWorkspaceName=$logAnalyticsWorkspaceName \
 logAnalyticsWorkspaceRegion=$logAnalyticsWorkspaceRegion
The output of the deployment is written to the console once complete. Search the output for the following JSON:

"outputs": {
        "cosmosDb": {
          "type": "Object",
          "value": {
            "hostName": <value>,
            "secret": <value>,
            "username": <value>
          }
        },
        "eventHubs": {
          "type": "Object",
          "value": {
            "taxi-fare-eh": <value>,
            "taxi-ride-eh": <value>
          }
        },
        "logAnalytics": {
          "type": "Object",
          "value": {
            "secret": <value>,
            "workspaceId": <value>
          }
        }
},
These values are the secrets that will be added to Databricks secrets in upcoming sections. Keep them secure until you add them in those sections.

Add a Cassandra table to the Cosmos DB Account
In the Azure portal, navigate to the resource group created in the deploy the Azure resources section above. Click on Azure Cosmos DB Account. Create a table with the Cassandra API.

In the overview blade, click add table.

When the add table blade opens, enter newyorktaxi in the Keyspace name text box.

In the enter CQL command to create the table section, enter neighborhoodstats in the text box beside newyorktaxi.

In the text box below, enter the following:

(neighborhood text, window_end timestamp, number_of_rides bigint, total_fare_amount double, total_tip_amount double, average_fare_amount double, average_tip_amount double, primary key(neighborhood, window_end))
In the Table throughput section confirm that Autoscale is selected and that value 4000 is in the Table Max RU/s text box.

Click OK.

Add the Databricks secrets using the Databricks CLI
Tip: Make sure you have authenticated your Databricks CLI configuration. The simplest method in bash is to run:

export DATABRICKS_AAD_TOKEN=$(az account get-access-token --resource 2ff814a6-3304-4ab8-85cb-cd0e6f879c1d | jq .accessToken --raw-output)
databricks configure --aad-token --host <enter Databricks Workspace URL from Portal>
The resource GUID (2ff814a6-3304-4ab8-85cb-cd0e6f879c1d) is a fixed value. For other options see Set up authentication in the Azure Databricks documentation. If you see a JSONDecodeError error when running a command, your token has exired and you can refresh by running the commands above again.

First, enter the secrets for EventHub:

Using the Azure Databricks CLI installed in step 4 of the prerequisites, create the Azure Databricks secret scope:

databricks secrets create-scope --scope "azure-databricks-job"
Add the secret for the taxi ride EventHub:

databricks secrets put --scope "azure-databricks-job" --key "taxi-ride"
Once executed, this command opens the vi editor. Enter the taxi-ride-eh value from the eventHubs output section in step 4 of the deploy the Azure resources section. Save and exit vi (if in edit mode hit ESC, then type ":wq").

Add the secret for the taxi fare EventHub:

databricks secrets put --scope "azure-databricks-job" --key "taxi-fare"
Once executed, this command opens the vi editor. Enter the taxi-fare-eh value from the eventHubs output section in step 4 of the deploy the Azure resources section. Save and exit vi (if in edit mode hit ESC, then type ":wq").

Next, enter the secrets for Cosmos DB:

Using the Azure Databricks CLI, add the secret for the Cosmos DB user name:

databricks secrets put --scope azure-databricks-job --key "cassandra-username"
Once executed, this command opens the vi editor. Enter the username value from the CosmosDb output section in step 4 of the deploy the Azure resources section. Save and exit vi (if in edit mode hit ESC, then type ":wq").

Next, add the secret for the Cosmos DB password:

databricks secrets put --scope azure-databricks-job --key "cassandra-password"
Once executed, this command opens the vi editor. Enter the secret value from the CosmosDb output section in step 4 of the deploy the Azure resources section. Save and exit vi (if in edit mode hit ESC, then type ":wq").

Note: If using an Azure Key Vault-backed secret scope, the scope must be named azure-databricks-job and the secrets must have the exact same names as those above.

Add the Census Neighborhoods data file to the Databricks file system
Create a directory in the Databricks file system:

dbfs mkdirs dbfs:/azure-databricks-job
Navigate to the DataFile folder and enter the following:

dbfs cp cb_2020_36_cousub_500k.zip dbfs:/azure-databricks-job/
Note: The filename may change if you obtain a shapefile for a different year.

Build the .jar files for the Databricks job
To build the jars using a docker container from a bash prompt change to the azure directory and run:

docker run -it --rm -v `pwd`:/streaming_azuredatabricks_azure -v ~/.m2:/root/.m2 maven:3.6.3-jdk-8 mvn -f /streaming_azuredatabricks_azure/pom.xml package
Note: Alternately, use your Java IDE to import the Maven project file named pom.xml located in the azure directory. Perform a clean build.

The outputs of the build is a file named azure-databricks-job-1.0-SNAPSHOT.jar in the ./AzureDataBricksJob/target directory.

Create a Databricks cluster
In the Databricks workspace, click Compute, then click Create cluster. Enter the cluster name you created in step 3 of the configure custom logging for the Databricks job section above.

Select Standard for Cluster Mode.

Set Databricks runtime version to 7.3 Extended Support (Scala 2.12, Apache Spark 3.0.1)

Deselect Enable autoscaling.

Set Worker Type to Standard_DS3_v2.

Set Workers to 2.

Set Driver Type to Same as worker

Optional - Configure Azure Log Analytics
Follow the instructions in Monitoring Azure Databricks to build the monitoring library and upload the resulting library files to your workspace.

Click on Advanced Options then Init Scripts.

Enter dbfs:/databricks/spark-monitoring/spark-monitoring.sh.

Click the Add button.

Click the Create Cluster button.

Install dependent libraries on cluster
In the Databricks user interface, click on the home button.

Click on Compute in the navigtation menu on the left then click on the cluster you created in the Create a Databricks cluster step.

Click on Libraries, then click Install New.

In the Library Source control, select Maven.

Under the Maven Coordinates text box, enter com.microsoft.azure:azure-eventhubs-spark_2.12:2.3.21.

Select Install.

Repeat steps 3 - 6 for the com.datastax.spark:spark-cassandra-connector-assembly_2.12:3.0.1 Maven coordinate.

Repeat steps 3 - 5 for the org.geotools:gt-shapefile:23.0 Maven coordinate.

Enter https://repo.osgeo.org/repository/release/ in the Repository text box.

Click Install.

Create a Databricks job
Copy the azure-databricks-job-1.0-SNAPSHOT.jar file to the Databricks file system by entering the following command in the Databricks CLI:

databricks fs cp --overwrite AzureDataBricksJob/target/azure-databricks-job-1.0-SNAPSHOT.jar dbfs:/azure-databricks-job/
In the Databricks workspace, click "Jobs", "create job".

Enter a job name.

In the Task area, change Type to JAR and Enter com.microsoft.pnp.TaxiCabReader in the Main Class field.

Under Dependent Libraries click Add, this opens the Add dependent library dialog box.

Change Library Source to DBFS/ADLS, confirm that Library Type is Jar and enter dbfs:/azure-databricks-job/azure-databricks-job-1.0-SNAPSHOT.jar in the File Path text box and select Add.

In the Parameters field, enter the following (replace <Cosmos DB Cassandra host name> with a value from above):

["-n","jar:file:/dbfs/azure-databricks-job/cb_2020_36_cousub_500k.zip!/cb_2020_36_cousub_500k.shp","--taxi-ride-consumer-group","taxi-ride-eh-cg","--taxi-fare-consumer-group","taxi-fare-eh-cg","--window-interval","1 hour","--cassandra-host","<Cosmos DB Cassandra host name>"]
Under Cluster, click the drop down arrow and select the cluster created the Create a Databricks cluster section.

Click Create

Select the Runs tab and click Run Now.

Run the data generator
Navigate to the directory onprem in the GitHub repository.

cd ../onprem
Update the values in the file main.env as follows:

RIDE_EVENT_HUB=[Connection string for the taxi-ride event hub]
FARE_EVENT_HUB=[Connection string for the taxi-fare event hub]
RIDE_DATA_FILE_PATH=/DataFile/FOIL2013
MINUTES_TO_LEAD=0
PUSH_RIDE_DATA_FIRST=false
The connection string for the taxi-ride event hub is the taxi-ride-eh value from the eventHubs output section in step 4 of the deploy the Azure resources section. The connection string for the taxi-fare event hub the taxi-fare-eh value from the eventHubs output section in step 4 of the deploy the Azure resources section.

Run the following command to build the Docker image.

docker build --no-cache -t dataloader .
Navigate back to the repository root directory.

cd ..
Run the following command to run the Docker image.

docker run -v `pwd`/DataFile:/DataFile --env-file=onprem/main.env dataloader:latest
The output should look like the following:

Created 10000 records for TaxiFare
Created 10000 records for TaxiRide
Created 20000 records for TaxiFare
Created 20000 records for TaxiRide
Created 30000 records for TaxiFare
...
Hit CTRL+C to cancel the generation of data.

Verify the solution is running
To verify the Databricks job is running correctly, open the Azure portal and navigate to the Cosmos DB database. Open the Data Explorer blade and examine the data in the neighborhoodstats table, you should see results similar to:

average_fare _amount	average_tip _amount	neighborhood	number_of_rides	total_fare _amount	total_tip _amount	window_end
10.5	1.0	Bronx	1	10.5	1.0	1/1/2013 8:02:00 AM +00:00
12.67	2.6	Brooklyn	3	38	7.8	1/1/2013 8:02:00 AM +00:00
14.98	0.73	Manhattan	52	779	37.83	1/1/2013 8:02:00 AM +00:00
