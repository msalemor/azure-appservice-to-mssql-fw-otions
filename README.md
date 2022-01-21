# App Service to Azure SQL connectivity using IP lists, Service Endpoint and Private Endpoint

## App Service to Azure SQL with Outbound IPs firewall rule

Steps to configure this setup, from the portal:

- On the WebApp
  - Go to networking settings
  - Get the list of outbound IPs
- Navigate to the Azure SQL instance
  - Open the firewall settings
  - Add the list of the WebApp outbound IPs

Other Options:

- You could use a NAT Gateway to use get and use just one IP from the WebApp on the Azure SQL firewall settings

Routing:

- Traffic will be routed out to the Internet into the Paas service (in this case the Azure SQL)
- Only traffic from the allowed IPs will be allowed to ingress the Azure SQL defined on the firewall rules

Automation:
- This code can add the WebApp outbound IPs to the Azure SQL programmatically

```bash
#!/bin/bash  

# Set variables
RG=<RG GROUP NAME>
WAPP=<WEBAPP NAME>
DBSVR=<MSSQL SVR NAME>

# Get the IPS
IPS=$(az webapp show -n $WAPP -g $RG --query "outboundIpAddresses" -o tsv)

if [ IPS != "" ];
then
    # remove the trailing \r
    IPS=${IPS::-1} 

    # Split IPs into an array
    IFS=','
    read -ra OIPS <<< "$IPS"

    c=1
    RULE_NAME="rule for $WAPP $c"
        
    for i in "${OIPS[@]}";   
    do  
        echo "$RULE_NAME : $i"        
        az sql server firewall-rule create -g $RG -s $DBSVR -n $RULE_NAME --start-ip-address $i --end-ip-address $i
        ((c=c+1))
        RULE_NAME="rule for $WAPP $c"        
    done  
fi
```

## Service Endpoint vs Private Link vs App Service Environment

If you have an App Service Environment but aren't using SQL Managed Instance, you can still use a Private Endpoint for private connectivity to a SQL Database. If you already have SQL Managed Instance but are using multi-tenant App Service, you can still use regional VNet Integration to connect to the SQL Managed Instance private address.

You can use a Service Endpoint to secure the database. With a Service Endpoint, the private endpoint, PrivateLinkSubnet, and configuring the Route All regional VNet integration setting are unnecessary. You still need regional VNet Integration to route incoming traffic through the Virtual Network.

Compared to Service Endpoints, a Private Endpoint provides a private, dedicated IP address toward a specific instance, for example a logical SQL Server, rather than an entire service. Private Endpoints can help prevent data exfiltration towards other database servers. For more information, see Comparison between Service Endpoints and Private Endpoints.

An alternative approach for private connectivity is an App Service Environment for hosting the web application within an isolated environment, and Azure SQL Managed Instance as the database engine. Both of these services are natively deployed within a Virtual Network, so there's no need for VNet Integration or private endpoints. These offerings are typically more costly, because they provide single-tenant isolated deployment and other features.

Reference Documents:
- [Preivate WebApp](https://docs.microsoft.com/en-us/azure/architecture/example-scenario/private-web-app/private-web-app#alternatives)
- [Compare Service Endpoint vs Private Endpoint](https://docs.microsoft.com/en-us/azure/virtual-network/vnet-integration-for-azure-services#compare-private-endpoints-and-service-endpoints)

## App Service to Azure SQL with VNET Integration using Service Endpoint

Steps to configure this setup, from the portal:

- Create VNET
- Create a database subnet in the VNET
  - Enable Service Endpoint for Microsoft.SQL on the database subnet
- Enable VNET Integration on the App Service and point it to the database subnet
  - > Note: Make sure to enable ```Route all``` on the network configuration settings for the WebApp. This setting ensures that traffic is routed through the Azure backbone.
- Make a call to the database from the App Service using the public URI


Routing: 
- Traffic is routed from the App Service over the subnet and only traffic from that subnet will be allowed by the Azure SQL. 
- There’s no private IP assigned to the database. 
- If you dont open a public IP in the Azure SQL firewall settings, a jumpbox is needed to administer the database.

## App Service to Azure SQL with VNET Integration using Private Endpoint

Steps to configure this setup, from the portal:
- Create VNET
- Create a connection subnet
- Create a database subnet
- Create a SQL Private Endpoint and use database subnet
- Enable VNET integration and point the connection the connection subnet
  - > Note: Make sure to enable ```Route all``` on the network configuration settings for the WebApp. This setting ensures that traffic is routed through the Azure backbone.
- Make a call to the database from the App Service using the regular public URI

Routing:
- Traffic is routed from the App Service over the connection subnet to the database subnet and only traffic from that coming from the connection and database subnets is allowed to reach the database.
- The database gets a private IP assignment. These disables all other firewall rules.
- If you dont open a public IP in the Azure SQL firewall settings, a jumpbox is needed to administer the database. 
- NSGs can be applied between the connection and the database subnets to restrict traffic further.
- It is possible to have on-prem devices access the Azure SQL using a hybrid DNS configuration
