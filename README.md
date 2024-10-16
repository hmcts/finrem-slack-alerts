# Financial Remedy Slack Alerts

### A serverless Azure Function for Application Insights monitoring and alerts
This project was forked from [et-slack-alerts](https://github.com/hmcts/et-slack-alerts)

## Overview

This is a timer-trigger based Azure Function App written in Python to monitor an Azure-based application for any given event (in our case, we focused on exceptions). If events have occurred, it will send alerts to a given slack channel. 

### Functionality
The function is scheduled to run every 5 minutes (customisable) and performs the following tasks:
- Authenticates with an Azure Key Vault to retrieve relevant environment variables.
- Queries application insights to capture all log entries returned for a given query and timescale (both customisable).
- Filters unique operations (some log entries cover multiple operations, which can clutter up the returned logs.)
- Sends a second query to application insights to get the entire log history of a given operation.
- Builds a slack message containing a formatted table of unique event triggering operations in the given timeframe, with generated inline links to the relevant log histories.
- Sends a slack alert (via an environment variable-defined webhook url)

### Prerequisites
- [Azure Functions Core Tools](https://learn.microsoft.com/en-us/azure/azure-functions/functions-run-local?tabs=macos%2Cisolated-process%2Cnode-v4%2Cpython-v2%2Chttp-trigger%2Ccontainer-apps&pivots=programming-language-csharp#install-the-azure-functions-core-tools)
- [Python 3.7+](https://www.python.org/downloads/)
- [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)
- [Azurite (for local development)](https://learn.microsoft.com/en-us/azure/storage/common/storage-use-azurite?tabs=npm%2Cblob-storage)
- An Azure account/subscription
- An [Azure Key Vault](https://azure.microsoft.com/en-gb/products/key-vault)
- An [Application Insights](https://learn.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview?tabs=net) instance you want to monitor

## Environment Variables
This function requires several environment variables (defined within the given keyvault)
- `api-key` - An API key for your given app insights instance. You can obtain one of these via the `API Access` section in the left hand side navigation of your Application Insights instance.
- `app-id` - The 'Instrumentation Key' of the Application Insights instance. This can be found in the top part of the Overview section.
- `slack-webhook-url` - A slack webhook URL for you to send messages to. For this part you will likely need to contact myself (@Danny on Slack) or a Slack administrator to get a custom slack 'app' set up. This is much more trivial than it sounds, a few clicks at most.
- `tenant-id` - Standard for the entire organisation.
- `resource-group-name` - The resource group name that the Application Insights instance is stored within.
- `app-insights-resource-name` - The name of the Application Insights instance.
- `subscription-id` - The subscription id that the Application Insights instance is stored within.

## Azure Installation
### Create Azure resources
```
az group create --name "finrem-slack-alerts" --location "uksouth"
 
az keyvault create --name "finrem-slack-alerts" --resource-group "finrem-slack-alerts"
 
az storage account create --name "finremslackalertsstorage" --location "uksouth" --resource-group "finrem-slack-alerts" --sku Standard_LRS
 
az functionapp create --resource-group "finrem-slack-alerts" --consumption-plan-location "uksouth" --runtime python --runtime-version 3.11 --functions-version 4 --name "finrem-slack-alerts" --os-type linux --storage-account "finremslackalertsstorage"
```

### Add Managed Identity to the function app
From the Azure portal:
- Navigate to the finrem-slack-alerts function app
- Settings -> Identity
- Select System Assigned
- Toggle Status on
- Save

### Add function app to the key vault
From the Azure portal:
- Navigate to the finrem-slack-alerts key vault
- Access policies
- Create

### Add tags to function app
From the Azure portal:
- Navigate to the finrem-slack-alerts function app
- Add tags

```
environment: staging
Application: financial-remedy
businessArea: CFT
ExpiresAfter: 3000-01-01
builtFrom: https://github.com/hmcts/finrem-slack-alerts
```

## Development
### Setup
#### Install Azurite
```
npm install -g azurite
```

#### Create local.settings.json file in alerts folder
```
{
    "IsEncrypted": false,
    "Values": {
      "FUNCTIONS_WORKER_RUNTIME": "python",
      "AzureWebJobsFeatureFlags": "EnableWorkerIndexing",
      "AzureWebJobsStorage": "UseDevelopmentStorage=true"
    }
}
```

#### Clone the repository
```
git clone https://github.com/hmcts/finrem-slack-alerts.git
```

#### Install dependencies
```
cd finrem-slack-alerts
<optionally install a virtual environment using e.g. venv>
python -m venv .venv
source .venv/bin/activate

pip install -r requirements.txt
```

### Run the function locally
First start Azurite in another tab
```
azurite --silent
```

Run the function
```
func start
```
