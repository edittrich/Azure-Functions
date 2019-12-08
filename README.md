# Links  
[Function Core Tools](https://docs.microsoft.com/en-us/azure/azure-functions/functions-run-local)    
[Sample](https://docs.microsoft.com/en-us/azure/azure-functions/functions-event-hub-cosmos-db)

# Install Function Core Tools  
npm install -g azure-functions-core-tools

# Login  
az login  
az account set --subscription edittrich

# Variables  
RESOURCE_GROUP=edittrich-data  
EVENT_HUB_NAMESPACE=edittrich-data-eh-ns  
EVENT_HUB_NAME=edittrich-data-eh-eh  
EVENT_HUB_AUTHORIZATION_RULE=edittrich-data-eh-ar  
COSMOS_DB_ACCOUNT=edittrich-data-cd-ac  
STORAGE_ACCOUNT=edittrichdatastac  
FUNCTION_APP=edittrich-data-fa  
LOCATION=westeurope

# Resource Group  
az group create \  
    --name $RESOURCE_GROUP \  
    --location $LOCATION

# Event Hub  
az eventhubs namespace create \  
    --resource-group $RESOURCE_GROUP \  
    --name $EVENT_HUB_NAMESPACE  
az eventhubs eventhub create \  
    --resource-group $RESOURCE_GROUP \  
    --name $EVENT_HUB_NAME \  
    --namespace-name $EVENT_HUB_NAMESPACE \  
    --message-retention 1  
az eventhubs eventhub authorization-rule create \  
    --resource-group $RESOURCE_GROUP \  
    --name $EVENT_HUB_AUTHORIZATION_RULE \  
    --eventhub-name $EVENT_HUB_NAME \  
    --namespace-name $EVENT_HUB_NAMESPACE \  
    --rights Listen Send

# Cosmos DB	  
az cosmosdb create \  
    --resource-group $RESOURCE_GROUP \  
    --name $COSMOS_DB_ACCOUNT  
az cosmosdb database create \  
    --resource-group-name $RESOURCE_GROUP \  
    --name $COSMOS_DB_ACCOUNT \  
    --db-name TelemetryDb  
az cosmosdb collection create \  
    --resource-group-name $RESOURCE_GROUP \  
    --name $COSMOS_DB_ACCOUNT \  
    --collection-name TelemetryInfo \  
    --db-name TelemetryDb \  
    --partition-key-path '/temperatureStatus'

# Storage Account	  
az storage account create \  
    --resource-group $RESOURCE_GROUP \  
    --name $STORAGE_ACCOUNT \  
    --sku Standard_LRS

# Function App	  
az functionapp create \  
    --resource-group $RESOURCE_GROUP \  
    --name $FUNCTION_APP \  
    --storage-account $STORAGE_ACCOUNT \  
    --consumption-plan-location $LOCATION \  
    --runtime java

# Connection Strings  
AZURE_WEB_JOBS_STORAGE=$( \  
    az storage account show-connection-string \  
        --name $STORAGE_ACCOUNT \  
        --query connectionString \  
        --output tsv)  
echo $AZURE_WEB_JOBS_STORAGE  
EVENT_HUB_CONNECTION_STRING=$( \  
    az eventhubs eventhub authorization-rule keys list \  
        --resource-group $RESOURCE_GROUP \  
        --name $EVENT_HUB_AUTHORIZATION_RULE \  
        --eventhub-name $EVENT_HUB_NAME \  
        --namespace-name $EVENT_HUB_NAMESPACE \  
        --query primaryConnectionString \  
        --output tsv)  
echo $EVENT_HUB_CONNECTION_STRING  
COSMOS_DB_CONNECTION_STRING=$( \  
    az cosmosdb keys list \  
        --resource-group $RESOURCE_GROUP \  
        --name $COSMOS_DB_ACCOUNT \  
        --type connection-strings \  
        --query connectionStrings[0].connectionString \  
        --output tsv)  
echo $COSMOS_DB_CONNECTION_STRING

# Function App Settings  
az functionapp config appsettings set \  
    --resource-group $RESOURCE_GROUP \  
    --name $FUNCTION_APP \  
    --settings \  
        AzureWebJobsStorage=$AZURE_WEB_JOBS_STORAGE \  
        EventHubConnectionString=$EVENT_HUB_CONNECTION_STRING \  
        CosmosDBConnectionString=$COSMOS_DB_CONNECTION_STRING

# Maven  
mvn archetype:generate --batch-mode \  
    -DarchetypeGroupId=com.microsoft.azure \  
    -DarchetypeArtifactId=azure-functions-archetype \  
    -DappName=$FUNCTION_APP \  
    -DresourceGroup=$RESOURCE_GROUP \  
    -DgroupId=de.edittrich.azure \  
    -DartifactId=functions

mvn archetype:generate --batch-mode -DarchetypeGroupId=com.microsoft.azure -DarchetypeArtifactId=azure-functions-archetype -DappName=edittrich-data-fa -DresourceGroup=edittrich-data -DgroupId=de.edittrich.azure -DartifactId=functions

# Function App Settings Local  
cd functions  
func azure functionapp fetch-app-settings $FUNCTION_APP

func azure functionapp fetch-app-settings edittrich-data-fa

# Build, run locally and deploy to Azure  
mvn clean package  
mvn azure-functions:run  
mvn azure-functions:deploy

# Delete  
az group delete --name $RESOURCE_GROUP  
