The following workshop was created for the Microsoft x Emergent IoT Workshop Series . Goal was to provide the participants a hands-on introduction to Azure Digital Twin.

# STEP 1: Set up environment

## Download required tools

Download VS Code: https://code.visualstudio.com/download

Download .NET 6.0: https://dotnet.microsoft.com/en-us/download

Download Git: https://git-scm.com/downloads

Go to the Azure Portal and create account:

- Students: [Azure for Students](https://azure.microsoft.com/en-us/free/students/)
    
- Non-students: [Free Azure Trial](https://azure.microsoft.com/en-gb/free/search/?OCID=AIDcmm3bvqzxp1_SEM_a12a261aff841e1a17f24211a7299a56:G:s&ef_id=a12a261aff841e1a17f24211a7299a56:G:s&msclkid=a12a261aff841e1a17f24211a7299a56)

## First set variables below

Open the Azure Portal and start a [bash session](https://learn.microsoft.com/en-us/azure/cloud-shell/quickstart?tabs=azurecli)

    location="westeurope" 

    resourceGroup=rgemergentd2$(shuf -i 10000-99999 -n 1)

    storageAccount=saemergentd2$(shuf -i 10000-99999 -n 1)

    functionApp=faemergentd2$(shuf -i 10000-99999 -n 1)

    dtName=dtemergentd2$(shuf -i 10000-99999 -n 1)

    iotHubName=iothubemergentd2$(shuf -i 10000-99999 -n 1)

    deviceName=deviceemergentd2$(shuf -i 10000-99999 -n 1)

    eventhubSub=ehsubemergentd2$(shuf -i 10000-99999 -n 1)

### Copy the output of the following command in a notepad or something similar in case the Bash session restarts, such that you don't lose the var names

    echo "resourceGroup=$resourceGroup"

    echo "storageAccount=$storageAccount"

    echo "functionApp=$functionApp"

    echo "dtName=$dtName"

    echo "iotHubName=$iotHubName"

    echo "deviceName=$deviceName"

    echo "eventhubSub=$eventhubSub"

## Copy paste commands below in bash session

    az extension add --name functionapp

    az extension add --name azure-iot

    az extension update --name azure-iot

    az group create --name $resourceGroup --location $location

    az iot hub create --name $iotHubName --resource-group $resourceGroup --sku S1 

    az storage account create --name $storageAccount --resource-group $resourceGroup --location $location --sku Standard_LRS --kind StorageV2

    key=$(az storage account keys list -g $resourceGroup -n $storageAccount --query "[0].value" -o tsv)

    az storage container create --account-name $storageAccount --name container1 --account-key $key

    az functionapp create --resource-group $resourceGroup --consumption-plan-location $location --runtime dotnet --functions-version 4 --name $functionApp --storage-account $storageAccount

    az dt create --dt-name $dtName --resource-group $resourceGroup --location $location 

## Assign Data Owner role to your user

    az ad signed-in-user show

### *Copy the 'id' value and replace the "id-value" in the next command with "id", don't forget to put it between " "*

    az dt role-assignment create --dt-name $dtName --assignee "id-value" --role "Azure Digital Twins Data Owner"

## Create models

    git clone https://github.com/claestom/msftemergentworkshop.git
    
    cd msftemergentworkshop/models

    az dt model create --dt-name $dtName --models computerroom3.json

    az dt model create --dt-name $dtName --models floor3.json

    az dt model create --dt-name $dtName --models groept.json

## Create twins

    az dt twin create  --dt-name $dtName --dtmi "dtmi:contosocom:DigitalTwins:GroepT;1" --twin-id GroepT

    az dt twin create  --dt-name $dtName --dtmi "dtmi:contosocom:DigitalTwins:Floor3;1" --twin-id Floor3 --properties "{\"nPeople\": 24, \"Humidity\":      48,\"Temperature\":20}"

    az dt twin create  --dt-name $dtName --dtmi "dtmi:contosocom:DigitalTwins:ComputerRoom3;1" --twin-id ComputerRoom3 --properties "{\"nPeople\": 17, \"Humidity\":      87,\"Temperature\":23}"

## Create relationships

    az dt twin relationship create -n $dtName --relationship-id rel1 --relationship contains --twin-id GroepT --target Floor3

    az dt twin relationship create -n $dtName --relationship-id rel4 --relationship provides --twin-id Floor3 --target ComputerRoom3    

# STEP 2: Configure IoT Hub and Azure Functions
## Create device in IoT Hub to connect to

    az iot hub device-identity create --hub-name $iotHubName --device-id $deviceName

    az iot hub device-identity connection-string show --hub-name $iotHubName --device-id $deviceName

--> Copy the 'connectionString'

    cd device

    vim pythondevice.py

    i

--> Paste the replace connectionString with the value you just copied and press 'esc' afterwards

    :wq

--> You should have returned to the original window

    /usr/bin/python3.9 -m pip install --upgrade pip

    pip install azure-iot-hub azure-iot-device

    python pythondevice.py

After you see some data appearing, this means the IoT device is working. CTRL-C to stop and proceed to STEP 3.

# STEP 3: Create and deploy Azure Functions

## Open local Command Prompt

    mkdir digitaltwin

    cd digitaltwin

    code .

### First install following extensions:

    Ctrl + shift + X

    Azure Tools

    Azure Account

### Sign in to Azure and create Function

Ctrl + shift + P

Search: Azure: Sign In 

    Shift + Alt + A

Click on the Azure symbol in the left, vertical blade and go to *Workspace* and click on *Create Function...* 

On top a pop-up will follow.

    1) Select *digitaltwin*

    2) C#

    3) .NET 6 LTS

    4) Azure Event Grid trigger

    5) Replace EventGridTrigger1 with digitaltwindemo

    6) Leave namespace like it is (enter)

    Ctrl + Shift + E

open *digitaltwindemo.cs*

### Download following packages by running the commands below in a terminal window inside VS Code

Open new terminal: Ctrl + SHift + ù

    dotnet add package Azure.DigitalTwins.Core --version 1.4.0

    dotnet add package Azure.Identity --version 1.8.2

    dotnet add package Microsoft.Azure.WebJobs.Extensions.EventGrid --version 3.2.1

### Add code to Function and deploy to the cloud

    Ctrl + Shift + E

Replace code with code inside *functioncode.py* (GitHub - https://github.com/claestom/msftemergentworkshop/tree/main/azure%20function)

    CTRL-S to save the changes

    Shift + Alt + A

Go to *Workspace* and click *Deploy...* and next *Deploy to Function App...*

    Pop-up will follow on top

    Select

    Select Subscription

    Select functionapp created

    Wait until deployment is completed (~1 minute)

### Configure settings of the Function and connect to the IoT Hub using the Azure Bash (check step 1 for tutorial on how to open a session)

    az functionapp identity assign --resource-group $resourceGroup --name $functionApp

    --> Copy the PrincipalId and past it between " " as the principal-ID in the command below

    az dt role-assignment create --dt-name $dtName --assignee "" --role "Azure Digital Twins Data Owner"

    az dt show --dt-name $dtName

    --> Copy the hostname and past it after https:// in the next command

    az functionapp config appsettings set --resource-group $resourceGroup --name $functionApp --settings "ADT_SERVICE_URL=https://"

    subId=$(az account show --query 'id' --output tsv)

    az provider register --namespace Microsoft.Web

    az provider register --namespace Microsoft.EventGrid

    az eventgrid event-subscription create --name $eventhubSub --event-delivery-schema eventgridschema --source-resource-id /subscriptions/$subId/resourceGroups/$resourceGroup/providers/Microsoft.Devices/IotHubs/$iotHubName --included-event-types Microsoft.Devices.DeviceTelemetry --endpoint-type azurefunction --endpoint /subscriptions/$subId/resourceGroups/$resourceGroup/providers/Microsoft.Web/sites/$functionApp/functions/IoTHubtoTwins

# STEP 4: Test E2E solution

## Open new Bash session

    cd msftemergentworkshop/device

    python pythondevice.py
    
Go to your Azure Digital Twins instance in the Azure Portal and open the *Azure Digital Twins Explorer (preview)*

# Learning resources

Oreilly: [Building Industrial Digital Twins](https://learning.oreilly.com/library/view/building-industrial-digital/9781839219078/?sso_link=yes&sso_link_from=Microsoft-Prod)

Oreilly: [Hands-On Azure Digital Twins](https://learning.oreilly.com/library/view/hands-on-azure-digital/9781801071383/?sso_link=yes&sso_link_from=Microsoft-Prod)

[Exam AZ-220: Microsoft Azure IoT Developer](https://learn.microsoft.com/en-us/certifications/exams/az-220)

[MS Learn](https://docs.microsoft.com/en-us/learn/paths/develop-azure-digital-twins/)

[Microsoft Documentation](https://learn.microsoft.com/en-us/azure/digital-twins/)

[Azure Digital Twins Hands-On lab](https://github.com/Azure-Samples/digital-twins-samples/tree/master/HandsOnLab)

[Product website](https://azure.microsoft.com/en-us/services/digital-twins/)

[Digital Twin Definition Language](https://github.com/Azure/opendigitaltwins-dtdl/blob/master/DTDL/v2/DTDL.v2.md)

[BUilding Digital Twins using Azure - The IoT Show](https://learn.microsoft.com/en-us/shows/internet-of-things-show/building-a-digital-twins-platform-with-azure-iot-services)

[A Flight into IoT - Digital Twin Example](https://www.youtube.com/watch?v=YcpAzAj-eRw&t=7440s)

[IT/OT Data Integration with Azure Digital Twins, Azure Data Explorer, and Azure Synapse
](https://learn.microsoft.com/en-us/shows/internet-of-things-show/itot-data-integration-with-azure-digital-twins-azure-data-explorer-and-azure-synapse)
