# AppService to MS SQL connectivity IP lists, private connection and private endpoint

## App Service to SQL with Outbound IPs fire rule


Steps to configure, from the portal:

- On the WebApp
  - Go to networking settings
  - Get the list of outbound IPs
- Navigate to the SQL Server
- Open the firewall settings
- Add the list of the WebApp outbound IPs

Other Options:

- You could use a NAT Gateway to use just one IP

Routing:

- Traffic will be routed out to the Internet
- Only traffic from the allowed IPs will be ingress the MS SQL

> **Note:** this code can add the Web App outbound IPs to the Azure SQL automatically

```bash
#!/bin/bash  
RG=<RG GROUP NAME>
WAPP=<WEBAPP NAME>
DBSVR=<MSSQL SVR NAME>
IPS=$(az webapp show -n $WAPP -g $RG --query "outboundIpAddresses" -o tsv)

if [ IPS != "" ];
then
    #echo $IPS    

    # Set space as the delimiter
    IPS=${IPS::-1} # remove the trailing \r
    echo $IPS
    IFS=','

    # Read the split words into an array
    # based on space delimiter
    read -ra OIPS <<< "$IPS"

    c=1
    RULE_NAME="rule for $WAPP $c"
    
    for i in "${OIPS[@]}"; #accessing each element of array  
    do  
        echo "$RULE_NAME : $i"        
        az sql server firewall-rule create -g $RG -s $DBSVR -n $RULE_NAME --start-ip-address $i --end-ip-address $i
        ((c=c+1))
        RULE_NAME="rule for $WAPP $c"        
    done  
fi
```


## App Service VNET Integration using Service Endpoint

## App Service VNET Integration using Private Endpoint

