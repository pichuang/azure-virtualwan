# Lab - Virtual WAN Scenario: Route traffic through an Azure Firewall spoke

## Intro

The goal of this lab is to demonstrate and validate the Azure Virtual WAN scenario to route traffic through an NVA (in this case, we will use an Azure Firewall instead of NVA), the same published by the vWAN official document [Scenario: Route traffic through an NVA](https://docs.microsoft.com/en-us/azure/virtual-wan/scenario-route-through-nva).

### Lab diagram

The lab uses the same amount of VNETs (eight total) and two regions with Hubs, and the remote connectivity to two branches using site-to-site VPN using and BGP. Below is a diagram of what you should expect to get deployed:

![net diagram](./media/networkdiagram.png)

### Components

- Two Virtual WAN Hubs in two different regions (default EastUS2 and WestUS2).
- Eight VNETs (Spoke 1 to 8) where:
    - Four VNETs (spoke 1, 2, 3, and 4) are connected directly to their respective vHUBs.
    - The other four (indirect spokes) spoke 5, 6, 7, and 8.
    - Transit between indirect  Spoke2 and Spoke4 with Azure Firewall instead of NVAs.
- Each VNET (except 2 and 4) has a Linux VM accessible from SSH (need to adjust NSG to allow access) or serial console.
- All Linux VMs include basic networking utilities such as: traceroute, tcptraceroute, hping3, nmap, curl.
    - For connectivity tests, you can use curl <"Destnation IP"> and the output should be the VM name.
    - All VMs have default username azureuser and password Msft123Msft123 (you can change it under parameters section).
- There's UDRs associated to the indirect spoke VNETs 5, 6, 7, 8 with default route 0/0 to their respective Azure Firewall spoke.
- Virtual WAN hubs have routes to Azure Firewall spokes using summaries routes (10.2.0.0/16 -> Spoke2conn, 10.4.0.0/16 -> Spoke4conn)
- Spoke2conn and Spoke4conn have specific routes 10.2.1.0/24 and 10.2.2.0/24 next hop to spoke 2 Azure Firewall IP and 10.4.1.0/24 and 10.4.2.0/24 next hop to spoke 4 Azure Firewall IP.
- The outcome of the lab will be full transit between all ends (all VMs can reach each other).
- Two remote branches emulated in Azure with Azure VPN Gateway on each site and S2S VPN using BGP to their respective vHUB. ASN 65010 is assigned to Branch 1 and ASN 65009 is assigned to Branch 2 while vHUBs VPN Gateways on both Hubs have fixed ASN 65515.

### Deploy this solution

The lab is also available in the above .azcli that you can rename as .sh (shell script) and execute. You can open [Azure Cloud Shell (Bash)](https://shell.azure.com) and run the following commands build the entire lab:

```bash
wget -O irazfw-deploy.sh https://raw.githubusercontent.com/dmauser/azure-virtualwan/main/inter-region-azfw/irazfw-deploy.azcli
chmod +xr irazfw-deploy.sh
./irazfw-deploy.sh 
```

**Note:** the provisioning process will take around 60 minutes to complete.

Alternatively (recommended), you can run step-by-step to get familiar with the provisioning process and the components deployed:

```bash
#!/bin/bash
# Reference: https://docs.microsoft.com/en-us/azure/virtual-wan/scenario-route-through-nva

# Pre-Requisites
echo validating pre-requisites
az extension add --name virtual-wan 
az extension add --name azure-firewall 
# or updating vWAN and AzFirewall CLI extensions
az extension update --name virtual-wan
az extension update --name azure-firewall 

# Parameters (make changes based on your requirements)
region1=eastus2
region2=westus2
rg=lab-vwan-irazfw
vwanname=vwan-irazfw
hub1name=hub1
hub2name=hub2
username=azureuser
password="Msft123Msft123" #Please change your password
vmsize=Standard_B1s

#Variables
mypip=$(curl -4 ifconfig.io -s)

# Creating rg
az group create -n $rg -l $region1 --output none

# Creating virtual wan
echo Creating vwan and both hubs...
az network vwan create -g $rg -n $vwanname --branch-to-branch-traffic true --location $region1 --type Standard --output none
az network vhub create -g $rg --name $hub1name --address-prefix 192.168.1.0/24 --vwan $vwanname --location $region1 --sku Standard --no-wait
az network vhub create -g $rg --name $hub2name --address-prefix 192.168.2.0/24 --vwan $vwanname --location $region2 --sku Standard --no-wait

echo Creating branches VNETs...
# Creating location1 branch virtual network
az network vnet create --address-prefixes 10.100.0.0/16 -n branch1 -g $rg -l $region1 --subnet-name main --subnet-prefixes 10.100.0.0/24 --output none
az network vnet subnet create -g $rg --vnet-name branch1 -n GatewaySubnet --address-prefixes 10.100.100.0/26 --output none

# Creating location2 branch virtual network
az network vnet create --address-prefixes 10.200.0.0/16 -n branch2 -g $rg -l $region2 --subnet-name main --subnet-prefixes 10.200.0.0/24 --output none
az network vnet subnet create -g $rg --vnet-name branch2 -n GatewaySubnet --address-prefixes 10.200.100.0/26 --output none

echo Creating spoke VNETs...
# Creating spokes virtual network
# Region1
az network vnet create --address-prefixes 10.1.0.0/24 -n spoke1 -g $rg -l $region1 --subnet-name main --subnet-prefixes 10.1.0.0/27 --output none
az network vnet create --address-prefixes 10.2.0.0/24 -n spoke2 -g $rg -l $region1 --subnet-name main --subnet-prefixes 10.2.0.0/27 --output none
az network vnet create --address-prefixes 10.2.1.0/24 -n spoke5 -g $rg -l $region1 --subnet-name main --subnet-prefixes 10.2.1.0/27 --output none
az network vnet create --address-prefixes 10.2.2.0/24 -n spoke6 -g $rg -l $region1 --subnet-name main --subnet-prefixes 10.2.2.0/24 --output none

# Region2
az network vnet create --address-prefixes 10.3.0.0/24 -n spoke3 -g $rg -l $region2 --subnet-name main --subnet-prefixes 10.3.0.0/27 --output none
az network vnet create --address-prefixes 10.4.0.0/24 -n spoke4 -g $rg -l $region2 --subnet-name main --subnet-prefixes 10.4.0.0/27 --output none
az network vnet create --address-prefixes 10.4.1.0/24 -n spoke7 -g $rg -l $region2 --subnet-name main --subnet-prefixes 10.4.1.0/27 --output none
az network vnet create --address-prefixes 10.4.2.0/24 -n spoke8 -g $rg -l $region2 --subnet-name main --subnet-prefixes 10.4.2.0/24 --output none

echo Creating VNET peerings...
# vnet peering from spoke 5 and spoke 6 to spoke2
az network vnet peering create -g $rg -n spoke2-to-spoke5 --vnet-name spoke2 --allow-vnet-access --allow-forwarded-traffic --remote-vnet $(az network vnet show -g $rg -n spoke5 --query id --out tsv) --output none
az network vnet peering create -g $rg -n spoke5-to-spoke2 --vnet-name spoke5 --allow-vnet-access --allow-forwarded-traffic --remote-vnet $(az network vnet show -g $rg -n spoke2  --query id --out tsv) --output none
az network vnet peering create -g $rg -n spoke2-to-spoke6 --vnet-name spoke2 --allow-vnet-access --allow-forwarded-traffic --remote-vnet $(az network vnet show -g $rg -n spoke6 --query id --out tsv) --output none 
az network vnet peering create -g $rg -n spoke6-to-spoke2 --vnet-name spoke6 --allow-vnet-access --allow-forwarded-traffic --remote-vnet $(az network vnet show -g $rg -n spoke2  --query id --out tsv) --output none

# vnet peering from spoke 7 and spoke 8 to spoke4
az network vnet peering create -g $rg -n spoke4-to-spoke7 --vnet-name spoke4 --allow-vnet-access --allow-forwarded-traffic --remote-vnet $(az network vnet show -g $rg -n spoke7 --query id --out tsv) --output none 
az network vnet peering create -g $rg -n spoke7-to-spoke4 --vnet-name spoke7 --allow-vnet-access --allow-forwarded-traffic --remote-vnet $(az network vnet show -g $rg -n spoke4  --query id --out tsv) --output none
az network vnet peering create -g $rg -n spoke4-to-spoke8 --vnet-name spoke4 --allow-vnet-access --allow-forwarded-traffic --remote-vnet $(az network vnet show -g $rg -n spoke8 --query id --out tsv) --output none 
az network vnet peering create -g $rg -n spoke8-to-spoke4 --vnet-name spoke8 --allow-vnet-access --allow-forwarded-traffic --remote-vnet $(az network vnet show -g $rg -n spoke4  --query id --out tsv) --output none

echo Creating VMs in both branches...
# Creating a VM in each branch spoke
az vm create -n branch1VM  -g $rg --image ubuntults --public-ip-sku Standard --size $vmsize -l $region1 --subnet main --vnet-name branch1 --admin-username $username --admin-password $password --nsg "" --no-wait
az vm create -n branch2VM  -g $rg --image ubuntults --public-ip-sku Standard --size $vmsize -l $region2 --subnet main --vnet-name branch2 --admin-username $username --admin-password $password --nsg "" --no-wait

echo Creating NSGs in both branches...
#Updating NSGs:
az network nsg create --resource-group $rg --name default-nsg-$region1 --location $region1 -o none
az network nsg create --resource-group $rg --name default-nsg-$region2 --location $region2 -o none
# Adding my home public IP to NSG for SSH access
az network nsg rule create -g $rg --nsg-name default-nsg-$region1 -n 'default-allow-ssh' --direction Inbound --priority 100 --source-address-prefixes $mypip --source-port-ranges '*' --destination-address-prefixes '*' --destination-port-ranges 22 --access Allow --protocol Tcp --description "Allow inbound SSH" --output none
az network nsg rule create -g $rg --nsg-name default-nsg-$region2 -n 'default-allow-ssh' --direction Inbound --priority 100 --source-address-prefixes $mypip --source-port-ranges '*' --destination-address-prefixes '*' --destination-port-ranges 22 --access Allow --protocol Tcp --description "Allow inbound SSH" --output none
# Associating NSG to the VNET subnets (Spokes and Branches)
az network vnet subnet update --id $(az network vnet list -g $rg --query '[?location==`'$region1'`].{id:subnets[0].id}' -o tsv) --network-security-group default-nsg-$region1 -o none
az network vnet subnet update --id $(az network vnet list -g $rg --query '[?location==`'$region2'`].{id:subnets[0].id}' -o tsv) --network-security-group default-nsg-$region2 -o none

echo Creating VPN Gateways in both branches...
# Creating pips for VPN GW's in each branch
az network public-ip create -n branch1-vpngw-pip -g $rg --location $region1 --sku Basic --output none
az network public-ip create -n branch2-vpngw-pip -g $rg --location $region2 --sku Basic --output none

# Creating VPN gateways
az network vnet-gateway create -n branch1-vpngw --public-ip-addresses branch1-vpngw-pip -g $rg --vnet branch1 --asn 65510 --gateway-type Vpn -l $region1 --sku VpnGw1 --vpn-gateway-generation Generation1 --no-wait 
az network vnet-gateway create -n branch2-vpngw --public-ip-addresses branch2-vpngw-pip -g $rg --vnet branch2 --asn 65509 --gateway-type Vpn -l $region2 --sku VpnGw1 --vpn-gateway-generation Generation1 --no-wait

echo Creating Spoke VMs...
# Creating a VM in each connected spoke
az vm create -n spoke1VM  -g $rg --image ubuntults --public-ip-sku Standard --size $vmsize -l $region1 --subnet main --vnet-name spoke1 --admin-username $username --admin-password $password --nsg "" --no-wait
az vm create -n spoke3VM  -g $rg --image ubuntults --public-ip-sku Standard --size $vmsize -l $region2 --subnet main --vnet-name spoke3 --admin-username $username --admin-password $password --nsg "" --no-wait

az vm create -n spoke5VM  -g $rg --image ubuntults --public-ip-sku Standard --size $vmsize -l $region1 --subnet main --vnet-name spoke5 --admin-username $username --admin-password $password --nsg "" --no-wait
az vm create -n spoke6VM  -g $rg --image ubuntults --public-ip-sku Standard --size $vmsize -l $region1 --subnet main --vnet-name spoke6 --admin-username $username --admin-password $password --nsg "" --no-wait

az vm create -n spoke7VM  -g $rg --image ubuntults --public-ip-sku Standard --size $vmsize -l $region2 --subnet main --vnet-name spoke7 --admin-username $username --admin-password $password --nsg "" --no-wait
az vm create -n spoke8VM  -g $rg --image ubuntults --public-ip-sku Standard --size $vmsize -l $region2 --subnet main --vnet-name spoke8 --admin-username $username --admin-password $password --nsg "" --no-wait

#Enablingboot diagnostics for all VMs in the resource group (Serial console)
#Creating Storage Account (boot diagnostics + serial console)
let "randomIdentifier1=$RANDOM*$RANDOM" 
az storage account create -n sc$randomIdentifier1 -g $rg -l $region1 --sku Standard_LRS -o none
let "randomIdentifier2=$RANDOM*$RANDOM" 
az storage account create -n sc$randomIdentifier2 -g $rg -l $region2 --sku Standard_LRS -o none
#Enablingboot diagnostics
stguri1=$(az storage account show -n sc$randomIdentifier1 -g $rg --query primaryEndpoints.blob -o tsv)
stguri2=$(az storage account show -n sc$randomIdentifier2 -g $rg --query primaryEndpoints.blob -o tsv)
az vm boot-diagnostics enable --storage $stguri1 --ids $(az vm list -g $rg --query '[?location==`'$region1'`].{id:id}' -o tsv) -o none
az vm boot-diagnostics enable --storage $stguri2 --ids $(az vm list -g $rg --query '[?location==`'$region2'`].{id:id}' -o tsv) -o none

### Installing tools for networking connectivity validation such as traceroute, tcptraceroute, iperf and others (check link below for more details) 
nettoolsuri="https://raw.githubusercontent.com/dmauser/azure-vm-net-tools/main/script/nettools.sh"
for vm in `az vm list -g $rg --query "[?storageProfile.imageReference.offer=='UbuntuServer'].name" -o tsv`
do
 az vm extension set \
 --resource-group $rg \
 --vm-name $vm \
 --name customScript \
 --publisher Microsoft.Azure.Extensions \
 --protected-settings "{\"fileUris\": [\"$nettoolsuri\"],\"commandToExecute\": \"./nettools.sh\"}" \
 --no-wait
done

echo Checking Hub1 provisioning status...
# Checking Hub1 provisioning and routing state 
prState=''
rtState=''
while [[ $prState != 'Succeeded' ]];
do
    prState=$(az network vhub show -g $rg -n $hub1name --query 'provisioningState' -o tsv)
    echo "$hub1name provisioningState="$prState
    sleep 5
done

while [[ $rtState != 'Provisioned' ]];
do
    rtState=$(az network vhub show -g $rg -n $hub1name --query 'routingState' -o tsv)
    echo "$hub1name routingState="$rtState
    sleep 5
done
echo Creating Hub1 VPN Gateway...
# Creating VPN gateways in each Hub1
az network vpn-gateway create -n $hub1name-vpngw -g $rg --location $region1 --vhub $hub1name --no-wait 

echo Checking Hub2 provisioning status...
# Checking Hub2 provisioning and routing state 
prState=''
rtState=''
while [[ $prState != 'Succeeded' ]];
do
    prState=$(az network vhub show -g $rg -n $hub2name --query 'provisioningState' -o tsv)
    echo "$hub2name provisioningState="$prState
    sleep 5
done

while [[ $rtState != 'Provisioned' ]];
do
    rtState=$(az network vhub show -g $rg -n $hub2name --query 'routingState' -o tsv)
    echo "$hub2name routingState="$rtState
    sleep 5
done

echo Creating Hub2 VPN Gateway...
# Creating VPN gateways in each Hub2
az network vpn-gateway create -n $hub2name-vpngw -g $rg --location $region2 --vhub $hub2name --no-wait

echo Deploying Azure Firewall...
# Deploy Azure Firewall on Spoke2 and Spoke 4
## Log Analytics workspace name. 
Workspacename1=AZFirewall-$region1-Logs 
Workspacename2=AZFirewall-$region2-Logs 

#Build Azure Firewall / Note this section takes few minutes to complete.
#Spoke 2
az network vnet subnet create -g $rg --vnet-name spoke2 -n AzureFirewallSubnet --address-prefixes 10.2.0.64/26 --output none
az network public-ip create --name spoke2-azfw-pip --resource-group $rg --location $region1 --allocation-method static --sku standard --output none
az network firewall create --name spoke2-azfw --resource-group $rg --location $region1 --output none
az network firewall ip-config create --firewall-name spoke2-azfw --name FW-config --public-ip-address spoke2-azfw-pip --resource-group $rg --vnet-name spoke2 --output none
az network firewall update --name spoke2-azfw --resource-group $rg --output none

#Spoke4
az network vnet subnet create -g $rg --vnet-name spoke4 -n AzureFirewallSubnet --address-prefixes 10.4.0.64/26 --output none
az network public-ip create --name spoke4-azfw-pip --resource-group $rg --location $region2 --allocation-method static --sku standard --output none
az network firewall create --name spoke4-azfw --resource-group $rg --location $region2 --output none
az network firewall ip-config create --firewall-name spoke4-azfw --name FW-config --public-ip-address spoke4-azfw-pip  --resource-group $rg --vnet-name spoke4 --output none
az network firewall update --name spoke4-azfw --resource-group $rg --output none

#Creating firewall rule to allow all traffic
#Spoke2-azfw
az network firewall network-rule create --resource-group $rg \
--firewall-name spoke2-azfw \
--collection-name azfw-rules \
--priority 1000 \
--action Allow \
--name Allow-All \
--protocols Any \
--source-addresses "*" \
--destination-addresses "*" \
--destination-ports "*" \
--output none
#spoke4-azfw
az network firewall network-rule create --resource-group $rg \
--firewall-name spoke4-azfw \
--collection-name azfw-rules \
--priority 1000 \
--action Allow \
--name Allow-All \
--protocols Any \
--source-addresses "*" \
--destination-addresses "*" \
--destination-ports "*" \
--output none

#Creating Log Analytics Workspaces
#Spoke2-azfw
az monitor log-analytics workspace create -g $rg --workspace-name $Workspacename1 --location $region1 --no-wait
#Spoke4-azfw
az monitor log-analytics workspace create -g $rg --workspace-name $Workspacename2 --location $region2 --no-wait

#EnablingAzure Firewall diagnostics
#Spoke2-azfw
az monitor diagnostic-settings create -n 'toLogAnalytics' \
--resource $(az network firewall show --name spoke2-azfw --resource-group $rg --query id -o tsv) \
--workspace $(az monitor log-analytics workspace show -g $rg --workspace-name $Workspacename1 --query id -o tsv) \
--logs '[{"category":"AzureFirewallApplicationRule","Enabled":true}, {"category":"AzureFirewallNetworkRule","Enabled":true}, {"category":"AzureFirewallDnsProxy","Enabled":true}]' \
--metrics '[{"category": "AllMetrics","enabled": true}]' \
--output none
#Spoke4-azfw
az monitor diagnostic-settings create -n 'toLogAnalytics' \
--resource $(az network firewall show --name spoke4-azfw --resource-group $rg --query id -o tsv) \
--workspace $(az monitor log-analytics workspace show -g $rg --workspace-name $Workspacename2 --query id -o tsv) \
--logs '[{"category":"AzureFirewallApplicationRule","Enabled":true}, {"category":"AzureFirewallNetworkRule","Enabled":true}, {"category":"AzureFirewallDnsProxy","Enabled":true}]' \
--metrics '[{"category": "AllMetrics","enabled": true}]' \
--output none

echo Updating indirect spoke UDRs to use Firewall as next hop...
#UDRs for Spoke 5 and 6
## Creating UDR + Disable BGP Propagation
az network route-table create --name RT-to-Spoke2-AzFW  --resource-group $rg --location $region1 --disable-bgp-route-propagation true --output none
## Default route to AzFW
az network route-table route create --resource-group $rg --name Default-to-AzFw --route-table-name RT-to-Spoke2-AzFW \
--address-prefix 0.0.0.0/0 \
--next-hop-type VirtualAppliance \
--next-hop-ip-address $(az network firewall show --name spoke2-azfw --resource-group $rg --query "ipConfigurations[].privateIpAddress" -o tsv) \
--output none
## Associated RT-Hub-to-AzFW to Spoke 5 and 6.
az network vnet subnet update -n main -g $rg --vnet-name spoke5 --route-table RT-to-Spoke2-AzFW --output none
az network vnet subnet update -n main -g $rg --vnet-name spoke6 --route-table RT-to-Spoke2-AzFW --output none

#UDRs for Spoke 7 and 8
## Creating UDR + Disable BGP Propagation
az network route-table create --name RT-to-Spoke4-AzFW  --resource-group $rg --location $region2 --disable-bgp-route-propagation true --output none
## Default route to AzFW
az network route-table route create --resource-group $rg --name Default-to-AzFw --route-table-name RT-to-Spoke4-AzFW \
--address-prefix 0.0.0.0/0 \
--next-hop-type VirtualAppliance \
--next-hop-ip-address $(az network firewall show --name spoke4-azfw --resource-group $rg --query "ipConfigurations[].privateIpAddress" -o tsv) \
--output none
## Associated RT-Hub-to-AzFW to Spoke 7 and 8.
az network vnet subnet update -n main -g $rg --vnet-name spoke7 --route-table RT-to-Spoke4-AzFW --output none
az network vnet subnet update -n main -g $rg --vnet-name spoke8 --route-table RT-to-Spoke4-AzFW --output none

echo Validating Branches VPN Gateways provisioning...
#Branches VPN Gateways provisioning status
prState=$(az network vnet-gateway show -g $rg -n branch1-vpngw --query provisioningState -o tsv)
if [[ $prState == 'Failed' ]];
then
    echo VPN Gateway is in fail state. Deleting and rebuilding.
    az network vnet-gateway delete -n branch1-vpngw -g $rg
    az network vnet-gateway create -n branch1-vpngw --public-ip-addresses branch1-vpngw-pip -g $rg --vnet branch1 --asn 65510 --gateway-type Vpn -l $region1 --sku VpnGw1 --vpn-gateway-generation Generation1 --no-wait 
    sleep 5
else
    prState=''
    while [[ $prState != 'Succeeded' ]];
    do
        prState=$(az network vnet-gateway show -g $rg -n branch1-vpngw --query provisioningState -o tsv)
        echo "branch1-vpngw provisioningState="$prState
        sleep 5
    done
fi

prState=$(az network vnet-gateway show -g $rg -n branch2-vpngw --query provisioningState -o tsv)
if [[ $prState == 'Failed' ]];
then
    echo VPN Gateway is in fail state. Deleting and rebuilding.
    az network vnet-gateway delete -n branch2-vpngw -g $rg
    az network vnet-gateway create -n branch2-vpngw --public-ip-addresses branch2-vpngw-pip -g $rg --vnet branch2 --asn 65509 --gateway-type Vpn -l $region2 --sku VpnGw1 --vpn-gateway-generation Generation1 --no-wait 
    sleep 5
else
    prState=''
    while [[ $prState != 'Succeeded' ]];
    do
        prState=$(az network vnet-gateway show -g $rg -n branch2-vpngw --query provisioningState -o tsv)
        echo "branch2-vpngw provisioningState="$prState
        sleep 5
    done
fi

echo Validating vHubs VPN Gateways provisioning...
#vWAN Hubs VPN Gateway Status
prState=$(az network vpn-gateway show -g $rg -n $hub1name-vpngw --query provisioningState -o tsv)
if [[ $prState == 'Failed' ]];
then
    echo VPN Gateway is in fail state. Deleting and rebuilding.
    az network vpn-gateway delete -n $hub1name-vpngw -g $rg
    az network vpn-gateway create -n $hub1name-vpngw -g $rg --location $region1 --vhub $hub1name --no-wait
    sleep 5
else
    prState=''
    while [[ $prState != 'Succeeded' ]];
    do
        prState=$(az network vpn-gateway show -g $rg -n $hub1name-vpngw --query provisioningState -o tsv)
        echo $hub1name-vpngw "provisioningState="$prState
        sleep 5
    done
fi

prState=$(az network vpn-gateway show -g $rg -n $hub2name-vpngw --query provisioningState -o tsv)
if [[ $prState == 'Failed' ]];
then
    echo VPN Gateway is in fail state. Deleting and rebuilding.
    az network vpn-gateway delete -n $hub2name-vpngw -g $rg
    az network vpn-gateway create -n $hub2name-vpngw -g $rg --location $region2 --vhub $hub2name --no-wait
    sleep 5
else
    prState=''
    while [[ $prState != 'Succeeded' ]];
    do
        prState=$(az network vpn-gateway show -g $rg -n $hub2name-vpngw --query provisioningState -o tsv)
        echo $hub2name-vpngw "provisioningState="$prState
        sleep 5
    done
fi

echo Building VPN connections from VPN Gateways to the respective Branches...
# get bgp peering and public ip addresses of VPN GW and VWAN to set up connection
bgp1=$(az network vnet-gateway show -n branch1-vpngw -g $rg --query 'bgpSettings.bgpPeeringAddresses[0].defaultBgpIpAddresses[0]' -o tsv)
pip1=$(az network vnet-gateway show -n branch1-vpngw -g $rg --query 'bgpSettings.bgpPeeringAddresses[0].tunnelIpAddresses[0]' -o tsv)
vwanbgp1=$(az network vpn-gateway show -n $hub1name-vpngw -g $rg --query 'bgpSettings.bgpPeeringAddresses[0].defaultBgpIpAddresses[0]' -o tsv)
vwanpip1=$(az network vpn-gateway show -n $hub1name-vpngw -g $rg --query 'bgpSettings.bgpPeeringAddresses[0].tunnelIpAddresses[0]' -o tsv)

bgp2=$(az network vnet-gateway show -n branch2-vpngw -g $rg --query 'bgpSettings.bgpPeeringAddresses[0].defaultBgpIpAddresses[0]' -o tsv)
pip2=$(az network vnet-gateway show -n branch2-vpngw -g $rg --query 'bgpSettings.bgpPeeringAddresses[0].tunnelIpAddresses[0]' -o tsv)
vwanbgp2=$(az network vpn-gateway show -n $hub2name-vpngw -g $rg --query 'bgpSettings.bgpPeeringAddresses[0].defaultBgpIpAddresses[0]' -o tsv)
vwanpip2=$(az network vpn-gateway show -n $hub2name-vpngw  -g $rg --query 'bgpSettings.bgpPeeringAddresses[0].tunnelIpAddresses[0]' -o tsv)

# Creating virtual wan vpn site
az network vpn-site create --ip-address $pip1 -n site-branch1 -g $rg --asn 65510 --bgp-peering-address $bgp1 -l $region1 --virtual-wan $vwanname --device-model 'Azure' --device-vendor 'Microsoft' --link-speed '50' --with-link true --output none
az network vpn-site create --ip-address $pip2 -n site-branch2 -g $rg --asn 65509 --bgp-peering-address $bgp2 -l $region2 --virtual-wan $vwanname --device-model 'Azure' --device-vendor 'Microsoft' --link-speed '50' --with-link true --output none

# Creating virtual wan vpn connection
az network vpn-gateway connection create --gateway-name $hub1name-vpngw -n site-branch1-conn -g $rg --enable-bgp true --remote-vpn-site site-branch1 --internet-security --shared-key 'abc123' --output none
az network vpn-gateway connection create --gateway-name $hub2name-vpngw -n site-branch2-conn -g $rg --enable-bgp true --remote-vpn-site site-branch2 --internet-security --shared-key 'abc123' --output none

# Creating connection from vpn gw to local gateway and watch for connection succeeded
az network local-gateway create -g $rg -n site-$hub1name-LG --gateway-ip-address $vwanpip1 --asn 65515 --bgp-peering-address $vwanbgp1 -l $region1 --output none
az network vpn-connection create -n branch1-to-site-$hub1name -g $rg -l $region1 --vnet-gateway1 branch1-vpngw --local-gateway2 site-$hub1name-LG --enable-bgp --shared-key 'abc123' --output none

az network local-gateway create -g $rg -n site-$hub2name-LG --gateway-ip-address $vwanpip2 --asn 65515 --bgp-peering-address $vwanbgp2 -l $region2 --output none
az network vpn-connection create -n branch2-to-site-$hub2name -g $rg -l $region2 --vnet-gateway1 branch2-vpngw --local-gateway2 site-$hub2name-LG --enable-bgp --shared-key 'abc123' --output none

echo Configuring spoke1 and spoke3 vnet connection to their respective vHubs...

# **** Configure vWAN route default route table for send traffic to Azure Firewall: *****

# Creating spoke to vWAN connections to hub1
# Spoke1 vnet connection
az network vhub connection create -n spoke1conn --remote-vnet spoke1 -g $rg --vhub-name $hub1name --no-wait
# Spoke3 vnet connection
az network vhub connection create -n spoke3conn --remote-vnet spoke3 -g $rg --vhub-name $hub2name --no-wait

echo creating spoke 2 and spoke 4 vnet connections using static route to their respective Azure Firewalls...
#Spoke2 vnet connection and Static Route to Spoke2-azfw
az network vhub connection create -n spoke2conn --remote-vnet spoke2 -g $rg \
 --vhub-name $hub1name \
 --route-name $hub1name-indirect-spokes-rt \
 --address-prefixes 10.2.1.0/24 10.2.2.0/24 \
 --next-hop $(az network firewall show --name spoke2-azfw --resource-group $rg --query "ipConfigurations[].privateIpAddress" -o tsv) \
 --no-wait

prState=''
while [[ $prState != 'Succeeded' ]];
do
    prState=$(az network vhub connection show -n spoke2conn --vhub-name $hub1name -g $rg  --query 'provisioningState' -o tsv)
    echo "vnet connection spoke2conn provisioningState="$prState
    sleep 5
done

#Spoke4 vnet connection and Static Route to Spoke2-azfw
az network vhub connection create -n spoke4conn --remote-vnet spoke4 -g $rg \
 --vhub-name $hub2name \
 --route-name $hub2name-indirect-spokes-rt \
 --address-prefixes 10.4.1.0/24 10.4.2.0/24 \
 --next-hop $(az network firewall show --name spoke4-azfw --resource-group $rg --query "ipConfigurations[].privateIpAddress" -o tsv) \
 --no-wait

prState=''
while [[ $prState != 'Succeeded' ]];
do
    prState=$(az network vhub connection show -n spoke4conn --vhub-name $hub2name -g $rg  --query 'provisioningState' -o tsv)
    echo "vnet connection spoke4conn provisioningState="$prState
    sleep 5
done

echo Adding static routes in the Hub1 default route table to the indirect spokes via Azure Firewall...
# Creating summary route to indirect spokes 5 and 6 via spoke2
az network vhub route-table route add --destination-type CIDR --resource-group $rg \
 --destinations 10.2.0.0/16 \
 --name defaultroutetable \
 --next-hop-type ResourceID \
 --next-hop $(az network vhub connection show --name spoke2conn --resource-group $rg --vhub-name $hub1name --query id -o tsv) \
 --vhub-name $hub1name \
 --route-name to-spoke2-azfw \
 --output none

# Creating summary route to indirect spokes 7 and 8 via spoke4
az network vhub route-table route add --destination-type CIDR --resource-group $rg \
 --destinations 10.4.0.0/16 \
 --name defaultroutetable \
 --next-hop-type ResourceID \
 --next-hop $(az network vhub connection show --name spoke4conn --resource-group $rg --vhub-name $hub2name --query id -o tsv) \
 --vhub-name $hub1name \
 --route-name to-spoke4-azfw \
 --no-wait

echo Adding static routes in the Hub3 default route table to the indirect spokes via Azure Firewall...
# Creating summary route to indirect spokes 7 and 8 via spoke4
az network vhub route-table route add --destination-type CIDR --resource-group $rg \
 --destinations 10.4.0.0/16 \
 --name defaultroutetable \
 --next-hop-type ResourceID \
 --next-hop $(az network vhub connection show --name spoke4conn --resource-group $rg --vhub-name $hub2name --query id -o tsv) \
 --vhub-name $hub2name \
 --route-name to-spoke4-azfw \
 --output none
# Creating summary route to indirect spokes 5 and 6 via spoke2
az network vhub route-table route add --destination-type CIDR --resource-group $rg \
 --destinations 10.2.0.0/16 \
 --name defaultroutetable \
 --next-hop-type ResourceID \
 --next-hop $(az network vhub connection show --name spoke2conn --resource-group $rg --vhub-name $hub1name --query id -o tsv) \
 --vhub-name $hub2name \
 --route-name to-spoke2-azfw \
 --no-wait
echo Deployment has finished
```

### Validation

```bash
# Parameters 
rg=lab-vwan-irazfw #set resource group

#### Validate connectivity between VNETs and Branches


# 1) Test connectivity between VMs (they can be accessible via SSH over Public IP or Serial Console)

#List of VMs Public IP for SSH access
az network public-ip list -g $rg --query "[].{name:name,ip:ipAddress}" -o table 

#List of VMs Private IPs
for nicname in `az network nic list -g $rg --query [].name -o tsv`
do 
echo -e $nicname private IP:
az network nic show -g $rg --name $nicname --query "ipConfigurations[].privateIpAddress" --output tsv
echo -e 
done

# For connectivity test you can also use curl or traceroute. For example from BranchVM1 or BranchVM2 run:
curl 10.2.1.4 #expected output spoke5VM
traceroute 10.2.1.4
ping 10.2.1.4 -O

# 2) Dump VMs route tables:
for nicname in `az network nic list -g $rg --query [].name -o tsv`
do 
echo -e $nicname effective routes:
az network nic show-effective-route-table -g $rg --name $nicname --output table
echo -e 
done

# 3) Dump all vHUBs route tables.
for vhubname in `az network vhub list -g $rg --query "[].id" -o tsv | rev | cut -d'/' -f1 | rev`
do
  for routetable in `az network vhub route-table list --vhub-name $vhubname -g $rg --query "[].id" -o tsv`
   do
   if [ "$(echo $routetable | rev | cut -d'/' -f1 | rev)" != "noneRouteTable" ]; then
     echo -e vHUB: $vhubname 
     echo -e Effective route table: $(echo $routetable | rev | cut -d'/' -f1 | rev)   
     az network vhub get-effective-routes -g $rg -n $vhubname \
     --resource-type RouteTable \
     --resource-id $routetable \
     --query "value[].{addressPrefixes:addressPrefixes[0], asPath:asPath, nextHopType:nextHopType}" \
     --output table
     echo
    fi
   done
done

# Dump the Branches VPN Gateway routes:
vpngws=$(az network vnet-gateway list -g $rg --query [].name -o tsv) 
array=($vpngws)
for vpngw in "${array[@]}"
 do 
 echo "*** $vpngw BGP peer status ***"
 az network vnet-gateway list-bgp-peer-status -g $rg -n $vpngw -o table
 echo "*** $vpngw BGP learned routes ***"
 az network vnet-gateway list-learned-routes -g $rg -n $vpngw -o table
 echo
done

# VPN Gateways advertised routes per BGP peer.
vpngws=$(az network vnet-gateway list -g $rg --query [].name -o tsv) 
array=($vpngws)
for vpngw in "${array[@]}"
 do 
  ips=$(az network vnet-gateway list-bgp-peer-status -g $rg -n $vpngw --query 'value[].{ip:neighbor}' -o tsv)
  array2=($ips)
   for ip in "${array2[@]}"
   do
   echo *** $vpngw advertised routes to peer $ip ***
   az network vnet-gateway list-advertised-routes -g $rg -n $vpngw -o table --peer $ip
  done
 echo
done

# Azure Firewall Rules 
# Over portal check Log Analytics and check the Firewall activities using Network Rule query.
```

### Clean-up

```bash
# Parameters 
rg=lab-vwan-irazfw #set resource group

### Clean up
az group delete -g $rg --no-wait 
```
