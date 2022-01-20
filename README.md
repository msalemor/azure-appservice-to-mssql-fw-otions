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

- You could use a NAT Gateway to use just one IP from the WebApp to the Azure SQL

Routing:

- Traffic will be routed out to the Internet
- Only traffic from the allowed IPs will be allowed to ingress the Azure SQL

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


## App Service with VNET Integration using Service Endpoint

Steps to configure this setup, from the portal:

- Create VNET
- Create a database subnet in the VNET
  - Enable Service Endpoint for Microsoft.SQL on the database subnet
- Enable VNET Integration on the App Service and point it to the database subnet
  - > Note: Make sure to enable ```Route all``` on the network configuration settings for the WebApp
- Make a call to the database from the App Service using the public URI


Routing: 
- Traffic is routed from the App Service over the subnet and only traffic from that subnet will be allowed by the Azure SQL. 
- There’s no private IP assigned to the database. 
- A jumpbox is needed to administer the database.

## App Service with VNET Integration using Private Endpoint

Steps to configure this setup, from the portal:
- Create VNET
- Create a connection subnet
- Create a database subnet
- Create a SQL Private Endpoint and use database subnet
- Enable VNET integration and point the connection the connection subnet
  - > Note: Make sure to enable ```Route all``` on the network configuration settings for the WebApp
- Make a call to the database from the App Service using the regular public URI

Routing:
- Traffic is routed from the App Service over the connection subnet to the database subnet and only traffic from that coming from the connection and database subnets is allowed to reach the database.
- The database gets a private IP assignment. These disables all other firewall rules.
- A jumpbox is needed to administer the database. 
- NSGs can be applied between the connection and the database subnet to restrict traffic further.
- If is possible to have on-prem devices access the Azure SQL using hybrid DNS configuration
