# AVS Network Scenario 2: Step-by-Step Guide
The following are step-by-step instructions on what needs to be deployed in your Hub vNET and AVS Transit vNET to successfully deploy AVS Network Scenario 2. I would recommend while following along all steps to have the network diagram up on a separate monitor.

https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/scenarios/azure-vmware/example-architectures

 ![image](https://user-images.githubusercontent.com/97964083/213535226-6701fe5d-cfab-4469-aa99-1093e39b790c.png)


**Hub vNET**

1.	Please create the following subnets in your Hub vNET. 

       ExternalFWSubnet: Recommend a minimum of /27 or larger. Name can be anything you’d like, just please make sure it indicates “External”.

       InsideFWSubnet: Recommend a minimum of /27 or larger. Name can be anything you’d like, just please make sure it indicates “Internal”.

       ManagementSubnet: Recommend a minimum of /27 or larger. Name can be anything you’d like, just please make sure it indicates “Management”.

       RouteServerSubnet: Must be a minimum of /27 or larger and the subnet name must be exactly RouteServerSubnet

2. Create an Azure Route Server. Please do this during a maintenance window because there will be a small outage between on-prem and Azure during the ARS deployment.

   https://learn.microsoft.com/en-us/azure/route-server/quickstart-configure-route-server-portal#create-a-route-server-1

3. Once Azure Route Server has been deployed, enable Branch-To-Branch.

   https://learn.microsoft.com/en-us/azure/route-server/quickstart-configure-route-server-portal#configure-route-exchange

4. Create your FW NVA which you can get in the Marketplace. In our example in the diagram, we show just one FW NVA. However, if you would like to have redundancy you should be deploying two FW NVA’s (Active/Standby, Active/Active). Make sure to use a private BGP AS that is not reserved in Azure. Please make sure that the BGP AS is different from the one you’re going to use on the BGP NVA in the AVS Transit vNET.
	![image](https://user-images.githubusercontent.com/97964083/213557308-97eb08c0-cb5a-40b1-9671-3e0ff09c2b0a.png)

 
5. Configure BGP routing on your FW NVA Internal IP to peer with the Azure Route Server. Make sure to use an eBGP multi-hop of 255 when peering from the FW NVA to Azure Route Server

    https://learn.microsoft.com/en-us/azure/route-server/quickstart-configure-route-server-portal#complete-the-configuration-on-the-nva

6. Configure BGP Peering from Azure Route Server to FW NVA Internal IP’s

    https://learn.microsoft.com/en-us/azure/route-server/quickstart-configure-route-server-portal#set-up-peering-with-nva

7. Confirm BGP Neighbor is up between FW NVA and Azure Route Server in Hub vNET. You can check this on the NVA side. 

8. Create a Route Table in the same region where your Hub vNET is deployed. 

   https://learn.microsoft.com/en-us/azure/virtual-network/manage-route-table#create-a-route-table

    Click “Yes” for “Propagate gateway routes”

    Create routes for the management AVS /22 block and AVS Segments to point to the next-hop of the FW Internal IP as shown in the diagram below

    Associate the Routing table with the GatewaySubnet in your Hub vNET.

    https://learn.microsoft.com/en-us/azure/virtual-network/manage-route-table#associate-a-route-table-to-a-subnet
    
    ![image](https://user-images.githubusercontent.com/97964083/213558220-68ac4bfd-38fc-408d-bc0e-1cfc2acfe5dc.png)


9. Your FW NVA’s have an interface in each of following subnets below. However, once a default-route is advertised from your FW NVA to your Azure Route Server (ARS), there is a potential for a routing loop to happen on both External and Management Subnets for your FW NVA. This routing loop only impacts traffic that is destined to the Internet. To avoid this loop, you will need to create a Routing-Table for both your External and Management Subnets only (Do Not Create One for Inside Subnet).

	In each routing table, you will create a default-route (0.0.0.0/0) User Defined Route (UDR) with a next-hop of “Internet”. Once the entry has been created, you can then associate the Routing-Table to the Subnet.

	ExternalFWSubnet: Associate the External Routing-Table to this Subnet

	ManagementSubnet: Associate the Management Routing-Table to this Subnet

	***InsideFWSubnet: DO NOT create a Routing Table for your inside subnet because it won’t be needed for this scenario.*** 
	
	https://learn.microsoft.com/en-us/azure/virtual-network/manage-route-table#create-a-route-table

	https://learn.microsoft.com/en-us/azure/virtual-network/manage-route-table#associate-a-route-table-to-a-subnet






**AVS Transit vNET**

6.	Create a new vNET that will be your “AVS Transit Virtual Network” in the same region where you will be deploying your AVS Private Cloud. Please make sure to size the vNET properly for a minimum of four subnets. 
a.	Subnets to deploy inside AVS Transit vNET
a.	RouteServerSubnet – Must be a minimum of /27 or larger and the subnet name must be exactly RouteServerSubnet
b.	GatewaySubnet – Must be a minimum of /27 or larger and the subnet name must be exactly GatewaySubnet
c.	InsideSubnet - Recommend a minimum of /27 or larger. Name can be anything you’d like, just please make sure it indicates “Inside”.
d.	OutsideSubnet - Recommend a minimum of /27 or larger. Name can be anything you’d like, just please make sure it indicates “Outside”.

7.	Create an ExpressRoute Virtual Network Gateway inside the AVS Transit vNET. Please see the link below on how to do the install. 
https://learn.microsoft.com/en-us/azure/expressroute/expressroute-howto-add-gateway-portal-resource-manager#create-the-virtual-network-gateway
We also recommend deploying a Zone-Redundant SKU
•	ErGw1AZ
•	ErGw2AZ
•	ErGw3AZ
https://learn.microsoft.com/en-us/azure/expressroute/expressroute-about-virtual-network-gateways#zrgw

8.	Create an Azure Route Server. You don’t need to complete this step during a maintenance window since this vNET is not connected back to on-prem via a Network Gateway. 
https://learn.microsoft.com/en-us/azure/route-server/quickstart-configure-route-server-portal#create-a-route-server-1
a.	Once Azure Route Server has been deployed, please enable Branch-To-Branch
                             https://learn.microsoft.com/en-us/azure/route-server/quickstart-configure-route-server-portal#configure-route-exchange

9.	Create your BGP NVA. For example, this can be a Cisco NVA (Cisco CSR) which you can get in the Marketplace. In our example in the diagram, we show just one BGP NVA. However, if you would like to have redundancy you should be deploying two BGP NVA’s. Please make sure you deploy the NVA with at least two NICs as shown in the diagram.

a.	One NIC will be connected to the InsideSubnet and the other will be connected to the OutsideSubnet.
b.	You would repeat the same steps above if you are deploying an additional BGP NVA.

10.	Configure a BGP neighbor between your BGP NVA Inside IP and Azure Route Server in the AVS Transit vNET.

a.	Make sure to use a private BGP AS that is not reserved by Azure. Please make sure that this AS is different from the one you’re using on your FW NVA.
 
b.	Configure BGP routing on your BGP NVA Inside IP to peer with the Azure Route Server
c.	Please make sure to use an eBGP multi-hop of 5 when peering from the BGP NVA to Azure Route Server
https://learn.microsoft.com/en-us/azure/route-server/quickstart-configure-route-server-portal#complete-the-configuration-on-the-nva

d.	Configure BGP Peering from Azure Route Server to BGP NVA Inside IP
https://learn.microsoft.com/en-us/azure/route-server/quickstart-configure-route-server-portal#set-up-peering-with-nva

e.	Confirm BGP Neighbor is up between the BGP NVA and Azure Route Server in AVS Transit vNET. You can check this on the NVA side. 

11.	Peer your Hub vNET with the AVS Transit vNET
https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-manage-peering#create-a-peering

12.	Create a Route Table in the same region where your AVS Transit vNET is deployed. 
https://learn.microsoft.com/en-us/azure/virtual-network/manage-route-table#create-a-route-table
a.	Click “No” for “Propagate gateway routes”
b.	Create routes for Hub vNET IP Space or any Spoke vNETs to point to the next-hop of the FW Internal IP as shown in the diagram below
c.	Create a default-route 0.0.0.0/0 with a next-hop of the FW Internal IP.
d.	Associate the Routing table with the “OutsideSubnet” in your AVS Transit vNET.
https://learn.microsoft.com/en-us/azure/virtual-network/manage-route-table#associate-a-route-table-to-a-subnet
 

13.	Configure BGP peering between the Outside Interface of the BGP NVA to the Azure Route Server in the Hub vNET. 
a.	Configure BGP routing on your BGP NVA Outside Interface IP to peer with the Azure Route Server
b.	Please make sure to use an eBGP multi-hop of 5 when peering from the BGP NVA to Azure Route Server
https://learn.microsoft.com/en-us/azure/route-server/quickstart-configure-route-server-portal#complete-the-configuration-on-the-nva

c.	Configure BGP Peering from Hub vNET Azure Route Server to BGP NVA Outside IP
https://learn.microsoft.com/en-us/azure/route-server/quickstart-configure-route-server-portal#set-up-peering-with-nva

d.	Confirm BGP Neighbor is up between the BGP NVA and Azure Route Server in Hub vNET. You can check this on the NVA side.


14.	The last step can be bit difficult to understand so I drew this out since a picture speaks a thousand words. By default, the Azure Route Servers are hard coded to use BGP AS 65515 and there is currently no way to modify that. This brings us a small challenge because by default eBGP will drop “Learned Routes” where it sees its own BGP AS because this is the loop prevention behavior. For example, in the diagram below the BGP NVA learns routes from the Azure Route Server on the left. The BGP NVA will then advertise those routes to the Azure Route Server on the right. The Azure Route Server on the right will drop the routes because it already sees itself in the BGP AS Path 65515.  To solve this problem, we need the BGP NVA to rewrite the AS 65515 so neither ARS on both sides sees itself in the AS Path and drops the routes. 

I’ve done this change many times before with Cisco CSR and there are two ways to do it.

You can use the BGP replace-as feature 
https://content.cisco.com/chapter.sjs?uri=/searchable/chapter/content/en/us/td/docs/ios-xml/ios/iproute_bgp/configuration/xe-17/irg-xe-17-book/bgp-replace-ASNs.html.xml

Or you can simply use a feature called AS-Override. Where “x.x.x.x” is the IP of the Azure Route Server. 
neighbor x.x.x.x as-override

Regardless these two options will remove the problem of routes getting dropped. Please let me know if you’re using Cisco and I can send you help configs I’ve used in my labs in the past. 

 ![image](https://user-images.githubusercontent.com/97964083/213533269-82efec30-1efb-495a-9821-2392e93d65b4.png)
