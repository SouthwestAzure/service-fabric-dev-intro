# Deploy a .NET application in a Windows container to Azure Service Fabric

This tutorial shows you how to containerize an existing ASP.NET application and package it as a Service Fabric application. Run the containers on a Service Fabric cluster running in Azure.

In this tutorial, you learn how to:

1. Containerize an existing application using Visual Studio

2. Create an Azure SQL database

3. Create an Azure container registry

4. Deploy a Service Fabric application to Azure


# Prerequisites

1. If you don't have an Azure subscription, create a free account

2. Install Docker CE for Windows so that you can run containers on Windows 10.

3. Install Service Fabric runtime version 6.2 or later and the Service Fabric SDK version 3.1 or later.

4. Install Visual Studio 2017 version 15.7 or later with the Azure development and ASP.NET and web development workloads.

5. Install Azure PowerShell


#Download and run Fabrikam Fiber CallCenter

Download the Fabrikam Fiber CallCenter sample application. Click the download archive link. From the sourceCode directory in the fabrikam.zip file, extract the sourceCode.zip file and then extract the VS2015 directory to your computer.

Verify that the Fabrikam Fiber CallCenter application builds and runs without error. Launch Visual Studio as an administrator and open the FabrikamFiber.CallCenter.sln file. Press F5 to debug and run the application.


# Fabrikam web sample

Containerize the application

Right-click the FabrikamFiber.Web project > Add > Container Orchestrator Support. Select Service Fabric as the container orchestrator and click OK.

Click Yes to switch Docker to Windows containers now.

A new project Service Fabric application project FabrikamFiber.CallCenterApplication is created in the solution. A Dockerfile is added to the existing FabrikamFiber.Web project. A PackageRoot directory is also added to the FabrikamFiber.Web project, which contains the service manifest and settings for the new FabrikamFiber.Web service.

The container is now ready to be built and packaged in a Service Fabric application. Once you have the container image built on your machine, you can push it to any container registry and pull it down to any host to run.


# Create an Azure SQL DB

When running the Fabrikam Fiber CallCenter application in production, the data needs to be persisted in a database. There is currently no way to guarantee persistent data in a container, therefore you cannot store production data in SQL Server in a container.

We recommend Azure SQL Database. To set up and run a managed SQL Server DB in Azure, run the following script. Modify the script variables as necessary. clientIP is the IP address of your development computer.

If you are behind a corporate firewall, the IP address of your development computer may not be IP address exposed to the internet. To verify that the database has the correct IP address for the firewall rule, go to the Azure portal and find your database in the SQL Databases section. Click on its name, then in the Overview section click "Set server firewall". "Client IP address" is the IP address of your development machine. Ensure that it matches the IP address in the "AllowClient" rule.

	$subscriptionID="<subscription ID>"

	# Sign in to your Azure account and select your subscription.
	Login-AzureRmAccount -SubscriptionId $subscriptionID 

	# The data center and resource name for your resources.
	$dbresourcegroupname = "fabrikam-fiber-db-group"
	$location = "southcentralus"

	# The logical server name: Use a random value or replace with your own value (do not capitalize).
	$servername = "fab-fiber-$(Get-Random)"

	# Set an admin login and password for your database.
	# The login information for the server.
	$adminlogin = "ServerAdmin"
	$password = "Password@123"

	# The IP address of your development computer that accesses the SQL DB.
	$clientIP = "24.18.117.76"

	# The database name.
	$databasename = "call-center-db"

	# Create a new resource group for your deployment and give it a name and a location.
	New-AzureRmResourceGroup -Name $dbresourcegroupname -Location $location

	# Create the SQL server.
	New-AzureRmSqlServer -ResourceGroupName $dbresourcegroupname `
        -ServerName $servername `
    	-Location $location `
    	-SqlAdministratorCredentials $(New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $adminlogin, $(ConvertTo-SecureString -String $password -AsPlainText -Force))

	# Create the firewall rule to allow your development computer to access the server.
	New-AzureRmSqlServerFirewallRule -ResourceGroupName $dbresourcegroupname `
    	-ServerName $servername `
    	-FirewallRuleName "AllowClient" -StartIpAddress $clientIP -EndIpAddress $clientIP

	# Creeate the database in the server.
	New-AzureRmSqlDatabase  -ResourceGroupName $dbresourcegroupname `
    	-ServerName $servername `
    	-DatabaseName $databasename `
    	-RequestedServiceObjectiveName "S0"

	Write-Host "Server name is $servername"


# Update the web config

Back in the FabrikamFiber.Web project, update the connection string in the web.config file, to point to the SQL Server in the container. Update the Server part of the connection string for the server created by the previous script.


<add name="FabrikamFiber-Express" connectionString="Server=tcp:fab-fiber-1300282665.database.windows.net,1433;Initial Catalog=call-center-db;Persist Security Info=False;User ID=ServerAdmin;Password=Password@123;MultipleActiveResultSets=False;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;" providerName="System.Data.SqlClient" />

<add name="FabrikamFiber-DataWarehouse" connectionString="Server=tcp:fab-fiber-1300282665.database.windows.net,1433;Initial Catalog=call-center-db;Persist Security Info=False;User ID=ServerAdmin;Password=Password@123;MultipleActiveResultSets=False;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;" providerName="System.Data.SqlClient" />

# Note

You can use any SQL Server you prefer for local debugging, as long as it is reachable from your host. However, localdb does not support container -> host communication. If you want to use a different SQL database when building a release build of your web application, add another connection string to your web.release.config file.


# Create a container registry

Now that the application runs locally, start preparing to deploy to Azure. Container images need to be stored in a container registry. Create an Azure container registry using the following script. The container registry name is visible by other Azure subscriptions, so it must be unique. Before deploying the application to Azure, you push the container image to this registry. When the application deploys to the cluster in Azure, the container image is pulled from this registry.


	$acrresourcegroupname = "fabrikam-acr-group"

	$location = "southcentralus"

	$registryname="fabrikamregistry$(Get-Random)"

	New-AzureRmResourceGroup -Name $acrresourcegroupname -Location $location

	$registry = New-AzureRMContainerRegistry -ResourceGroupName $acrresourcegroupname -Name $registryname -EnableAdminUser -Sku Basic


# Create a Service Fabric cluster on Azure

You should already have a Service Fabric cluster created in the earlier labs. If not, follow the link below to setup a cluster. 

https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-cluster-creation-via-portal


# Allow your application running in Azure to access the SQL DB

Next, you need to enable the application running in Azure to access the SQL DB. Create a virtual network service endpoint for the Service Fabric cluster and then create a rule to allow that endpoint to access the SQL DB.


	# Create a virtual network service endpoint

	$clusterresourcegroup = "fabrikamfiber.callcenterapplication_RG"

	$resource = Get-AzureRmResource -ResourceGroupName $clusterresourcegroup -ResourceType Microsoft.Network/virtualNetworks | Select-Object -first 1

	$vnetName = $resource.Name

	Write-Host 'Virtual network name: ' $vnetName

	# Get the virtual network by name.

	$vnet = Get-AzureRmVirtualNetwork `

 	-ResourceGroupName $clusterresourcegroup `

 	-Name $vnetName

	Write-Host "Get the subnet in the virtual network:"

	# Get the subnet, assume the first subnet contains the Service Fabric cluster.

	$subnet = Get-AzureRmVirtualNetworkSubnetConfig -VirtualNetwork $vnet | Select-Object -first 1

	$subnetName = $subnet.Name

	$subnetID = $subnet.Id

	$addressPrefix = $subnet.AddressPrefix

	Write-Host "Subnet name: " $subnetName " Address prefix: " $addressPrefix " ID: " $subnetID

	# Assign a Virtual Service endpoint 'Microsoft.Sql' to the subnet.

	$vnet = Set-AzureRmVirtualNetworkSubnetConfig `

 	-Name $subnetName `

 	-AddressPrefix  $addressPrefix `

 	-VirtualNetwork $vnet `

 	-ServiceEndpoint Microsoft.Sql | Set-AzureRmVirtualNetwork

	$vnet.Subnets[0].ServiceEndpoints; # Display the first endpoint.

	# Add a SQL DB firewall rule for the virtual network service endpoint

	$subnet = Get-AzureRmVirtualNetworkSubnetConfig `

 	-Name  $subnetName `

 	-VirtualNetwork $vnet;

	$VNetRuleName="ServiceFabricClusterVNetRule"

	$vnetRuleObject1 = New-AzureRmSqlServerVirtualNetworkRule `

 	-ResourceGroupName $dbresourcegroupname `

 	-ServerName  $servername `

 	-VirtualNetworkRuleName $VNetRuleName `

 	-VirtualNetworkSubnetId $subnetID;


# Deploy the application to Azure

Now that the application is ready, you can deploy it to the cluster in Azure directly from Visual Studio. In Solution Explorer, right-click the FabrikamFiber.CallCenterApplication application project and select Publish. In Connection Endpoint, select the endpoint of the cluster that you created previously. In Azure Container Registry, select the container registry that you created previously. Click Publish to deploy the application to the cluster in Azure.


Follow the deployment progress in the output window. When the application is deployed, open a browser and type in the cluster address and application port. For example, http://fabrikamfibercallcenter.southcentralus.cloudapp.azure.com:8659/.


