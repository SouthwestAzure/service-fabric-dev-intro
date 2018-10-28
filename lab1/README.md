# Service Fabric .NET Quickstart

This repository contains an quickstart project for [Microsoft Azure Service Fabric](https://azure.microsoft.com/services/service-fabric/). The quickstart project contains a single application with multiple services demonstrating the basic concepts of service communication and use of reliable dictionaries.

For a guided tour with the quickstart:
[Service Fabric .NET quickstart](https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-quickstart-dotnet)

More info on Service Fabric:

- [Documentation](https://docs.microsoft.com/azure/service-fabric/)
- [Service Fabric sample projects](https://azure.microsoft.com/resources/samples/?service=service-fabric)
- [Service Fabric open source home repo](https://github.com/azure/service-fabric)


## Task 1 - Create a Service Fabric Cluster in Azure

- You will only need a single Node Type - call it "Primary"
- Use default 5 node configuration
- Enable reverse proxy
- [Walkthrough](https://github.com/Microsoft/MCW-Microservices-architecture/blob/master/Hands-on%20lab/Before%20the%20HOL%20-%20Microservices%20architecture.md#task-1-provision-service-fabric-cluster)

## Task 2 - Install Service Fabric SDK for Visual Studio

- [Walkthrough](https://github.com/Microsoft/MCW-Microservices-architecture/blob/master/Hands-on%20lab/Before%20the%20HOL%20-%20Microservices%20architecture.md#task-5-install-service-fabric-sdk-for-visual-studio)
- Use latest version available

## Task 3 - Setup Certificate on Local Computer

- [Walkthrough](https://github.com/Microsoft/MCW-Microservices-architecture/blob/master/Hands-on%20lab/Before%20the%20HOL%20-%20Microservices%20architecture.md#task-6-setup-service-fabric-certificate)
- .pfx file password is blank/empty

## Task 4 - Clone Repository

- Setup SF cluster on local machine with a single node
- `git clone https://github.com/SouthwestAzure/service-fabric-dev-intro.git`

## Task 5 - Run and Debug Locally

- Run Visual Studio as Administrator
- Open [/lab1/Voting.sln](lab1/Voting.sln)
- Notice port 8080 specified in [/lab1/VotingWeb/PackageRoot/ServiceManifest.xml](/lab1/VotingWeb/PackageRoot/ServiceManifest.xml)
- F5 run from Visual Studio
- Browse [http://localhost:8080/](http://localhost:8080/)

## Task 6 - Deploy to Azure Cluster

- Use Cloud.xml parameters

## Task 7 - Allow Access via Load Balancer

- Setup load balancing probe (HTTP/8080)
- Setup load balancing port (TCP/8080)
- Browse http://**YOUR_CLUSTER**.**westus2**.cloudapp.azure.com:8080 (confirm name and region)
- [More Info](https://docs.microsoft.com/en-us/azure/service-fabric/create-load-balancer-rule)
