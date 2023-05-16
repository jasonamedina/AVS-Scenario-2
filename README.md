# AVS Network Scenario 2: Step-by-Step Guide
The following are step-by-step instructions on what needs to be deployed in your Hub vNET and AVS Transit vNET to successfully deploy AVS Network Scenario 2. I would recommend while following along all steps to have the network diagram up on a separate monitor.

**Example architectures for Azure VMWare Solutions**    
https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/scenarios/azure-vmware/example-architectures

 ![eslz-net-scenario-2](https://user-images.githubusercontent.com/97964083/213711299-98a4216e-ed77-41ea-8108-731d20559669.png)



**Hub vNET**

1. Please create the following subnets in your Hub vNET.   

   		ExternalFWSubnet: Recommend a minimum of /27 or larger. Name can be anything you’d like, just please make sure it indicates “External”.  
  		InsideFWSubnet: Recommend a minimum of /27 or larger. Name can be anything you’d like, just please make sure it indicates “Internal”.  
   		ManagementSubnet: Recommend a minimum of /27 or larger. Name can be anything you’d like, just please make sure it indicates “Management”.      
   		RouteServerSubnet: Must be a minimum of /27 or larger and the subnet name must be exactly RouteServerSubnet.  

2. Create an Azure Route Server. Please do this during a maintenance window because there will be a small outage between on-prem and Azure during the ARS deployment.

   **Create a Route Server**  
   https://learn.microsoft.com/en-us/azure/route-server/quickstart-configure-route-server-portal#create-a-route-server-1

3. Once Azure Route Server has been deployed, enable Branch-To-Branch.

   **Configure route exchange**  
   https://learn.microsoft.com/en-us/azure/route-server/quickstart-configure-route-server-portal#configure-route-exchange

4. Create your FW NVA which you can get in the Marketplace. In our example in the diagram, we show just one FW NVA. However, if you would like to have redundancy you should be deploying two FW NVA’s (Active/Standby, Active/Active). Make sure to use a private BGP AS that is not reserved in Azure. Please make sure that the BGP AS is different from the one you’re going to use on the BGP NVA in the AVS Transit vNET.
	![image](https://user-images.githubusercontent.com/97964083/213557308-97eb08c0-cb5a-40b1-9671-3e0ff09c2b0a.png)

 
5. Configure BGP routing on your FW NVA Internal IP to peer with the Azure Route Server. Make sure to use an eBGP multi-hop of 255 when peering from the FW NVA to Azure Route Server.

    **Complete the configuration on the NVA**  
    https://learn.microsoft.com/en-us/azure/route-server/quickstart-configure-route-server-portal#complete-the-configuration-on-the-nva

6. Configure BGP Peering from Azure Route Server in the Hub vNET to FW NVA Internal IP’s

    **Set up peering with NVA**  
    https://learn.microsoft.com/en-us/azure/route-server/quickstart-configure-route-server-portal#set-up-peering-with-nva

7. Confirm BGP Neighbor is up between FW NVA and Azure Route Server in Hub vNET. You can check this on the NVA side. 

8. Create a Route Table in the same region where your Hub vNET is deployed. 
    
    **Create a route table**  
   https://learn.microsoft.com/en-us/azure/virtual-network/manage-route-table#create-a-route-table

    Click “Yes” for “Propagate gateway routes”.  
    Create routes for the management AVS /22 block and AVS Segments to point to the next-hop of the FW Internal IP as shown in the diagram below.  
    Associate the Routing table with the GatewaySubnet in your Hub vNET.  

    **Associate a route table to a subnet**  
    https://learn.microsoft.com/en-us/azure/virtual-network/manage-route-table#associate-a-route-table-to-a-subnet
    
    ![image](https://user-images.githubusercontent.com/97964083/213558220-68ac4bfd-38fc-408d-bc0e-1cfc2acfe5dc.png)


9. Your FW NVA’s have an interface in each of following subnets below. However, once a default-route is advertised from your FW NVA to your Azure Route Server (ARS), there is a potential for a routing loop to happen on both External and Management Subnets for your FW NVA. This routing loop only impacts traffic that is destined to the Internet. To avoid this loop, you will need to create a Routing-Table for both your External and Management Subnets only (Do Not Create One for Inside Subnet).

	In each routing table, you will create a default-route (0.0.0.0/0) User Defined Route (UDR) with a next-hop of “Internet”. Once the entry has been created, you can then associate the Routing-Table to the Subnet.

	 	ExternalFWSubnet: Associate the External Routing-Table to this Subnet.  
		ManagementSubnet: Associate the Management Routing-Table to this Subnet.  
		
	 ***InsideFWSubnet: DO NOT create a Routing Table for your inside subnet because it won’t be needed for this scenario.***  
	
	**Create a route table**  
	https://learn.microsoft.com/en-us/azure/virtual-network/manage-route-table#create-a-route-table  
	
     **Associate a route table to a subnet**    
     https://learn.microsoft.com/en-us/azure/virtual-network/manage-route-table#associate-a-route-table-to-a-subnet  
 &nbsp;  
 &nbsp;  
 &nbsp;  
 &nbsp;  
 &nbsp;  
 &nbsp;  
 &nbsp;  
 &nbsp;  
 &nbsp;  
 &nbsp;  
 &nbsp;  
**AVS Transit vNET**

10.	Create a new vNET that will be your “AVS Transit Virtual Network” in the same region where you will be deploying your AVS Private Cloud. Please make sure to size the vNET properly for a minimum of four subnets. 

	Subnets to deploy inside AVS Transit vNET
	
		RouteServerSubnet: Must be a minimum of /27 or larger and the subnet name must be exactly RouteServerSubnet.  
		GatewaySubnet: Must be a minimum of /27 or larger and the subnet name must be exactly GatewaySubnet.  
		InsideSubnet: Recommend a minimum of /27 or larger. Name can be anything you’d like, just please make sure it indicates “Inside”.  
		OutsideSubnet: Recommend a minimum of /27 or larger. Name can be anything you’d like, just please make sure it indicates “Outside”.  

11. Peer your Hub vNET with the AVS Transit vNET

**Create a peering**  
https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-manage-peering#create-a-peering

12.	Create an ExpressRoute Virtual Network Gateway inside the AVS Transit vNET. Please see the link below on how to do the install.  

**Create the virtual network gateway**  
https://learn.microsoft.com/en-us/azure/expressroute/expressroute-howto-add-gateway-portal-resource-manager#create-the-virtual-network-gateway

We also recommend deploying a Zone-Redundant SKU

	ErGw1AZ  
	ErGw2AZ  
	ErGw3AZ  

**Zone-redundant gateway SKUs**  
https://learn.microsoft.com/en-us/azure/expressroute/expressroute-about-virtual-network-gateways#zrgw

13. Create an Azure Route Server. You don’t need to complete this step during a maintenance window since this vNET is not connected back to on-prem via a Network Gateway.

 **Create a Route Server**  
https://learn.microsoft.com/en-us/azure/route-server/quickstart-configure-route-server-portal#create-a-route-server-1

14. Once Azure Route Server has been deployed, please enable Branch-To-Branch

**Configure route exchange**  
https://learn.microsoft.com/en-us/azure/route-server/quickstart-configure-route-server-portal#configure-route-exchange

15. Create your BGP NVA. For example, this can be a Cisco NVA which you can get in the Marketplace. In our example in the diagram, we show just one BGP NVA. However, if you would like to have redundancy you should be deploying two BGP NVA’s. Please make sure you deploy the NVA with at least two NICs. 

16.	One NIC on the NVA will be connected to the InsideSubnet and the other will be connected to the OutsideSubnet.
	You would repeat the same steps above if you are deploying an additional BGP NVA.  
	![image](https://user-images.githubusercontent.com/97964083/213725429-2ee3bf4f-a31b-41fb-8ad8-57851e2478eb.png)

17.	Configure a BGP peer from your Azure Route Server in the AVS Transit vNET to peer with BGP NVA Inside IP.  
        
	 **Set up peering with NVA**  
         https://learn.microsoft.com/en-us/azure/route-server/quickstart-configure-route-server-portal#set-up-peering-with-nva
    
18.	Make sure to use a private BGP AS that is not reserved by Azure. 
	
 ![image](https://user-images.githubusercontent.com/97964083/213565242-fe68fca4-00e1-4a90-bbea-f15bdfd4aae0.png)
 
***Please make sure that the BGP AS used on the "BGP NVA" is different from the one you’re using on your "FW NVA".***

19. Configure BGP routing from your BGP NVA Inside IP to peer with the Azure Route Server in the AVS Transit vNET.  

 **Complete the configuration on the NVA**  
https://learn.microsoft.com/en-us/azure/route-server/quickstart-configure-route-server-portal#complete-the-configuration-on-the-nva

20. Configure eBGP multi-hop of 255 when peering from the BGP NVA to Azure Route Server. Please check with your vendor documentation on how to configure eBGP multi-hop. 

21. Configure BGP Peering from Azure Route Server Hub vNET to BGP NVA Outside IP

**Set up peering with NVA**  
  https://learn.microsoft.com/en-us/azure/route-server/quickstart-configure-route-server-portal#set-up-peering-with-nva

22. Confirm BGP Neighbor is up between the BGP NVA and Azure Route Server in both AVS Transit vNET and Hub vNET. You can check this on the NVA side. 

23. Create a Route Table in the same region where your AVS Transit vNET is deployed.
 

	Click “No” for “Propagate gateway routes”  
	Create routes for Hub vNET IP Space or any Spoke vNETs to point to the next-hop of the FW Internal IP as shown in the diagram below  
	Create a default-route 0.0.0.0/0 with a next-hop of the FW Internal IP.  
        Associate the Routing table with the “OutsideSubnet” in your AVS Transit vNET.  

 **Create a route table**  
https://learn.microsoft.com/en-us/azure/virtual-network/manage-route-table#create-a-route-table  

**Associate a route table to a subnet**  
https://learn.microsoft.com/en-us/azure/virtual-network/manage-route-table#associate-a-route-table-to-a-subnet  
 
24. Configure BGP from your BGP NVA Outside Interface IP to peer with the Azure Route Server in the Hub vNET.

25. Please make sure to use an eBGP multi-hop of 255 when peering from the BGP NVA to Azure Route Server. Please check with your vendor documentation on how to configure eBGP multi-hop. 

**Complete the configuration on the NVA**  
https://learn.microsoft.com/en-us/azure/route-server/quickstart-configure-route-server-portal#complete-the-configuration-on-the-nva

26. Configure BGP Peering from Hub vNET Azure Route Server to BGP NVA Outside IP

**Set up peering with NVA**  
https://learn.microsoft.com/en-us/azure/route-server/quickstart-configure-route-server-portal#set-up-peering-with-nva

27. Confirm BGP Neighbor is up between the BGP NVA and Azure Route Server in Hub vNET. You can check this on the NVA side.

28.  By default, the Azure Route Servers are hard coded to use BGP AS 65515 and up until the day of this writing there is currently no way to modify that. eBGP has a loop prevention mechanism where it will drops “BGP Routes” where it sees its own BGP AS in the "AS Path". For example, in the diagram below the BGP NVA learns routes from the Azure Route Server in the Hub vNET. The BGP NVA will then advertise those routes to the Azure Route Server in the AVS Transit vNET. The Azure Route Server in the AVS Transit vNET will drop the routes because it already sees itself in the BGP AS Path 65515.  To solve this problem, we need the BGP NVA to rewrite the AS 65515 so neither ARS on both vNETs sees itself in the AS Path and drops the routes. 

![image](https://user-images.githubusercontent.com/97964083/213778849-52b3a0e7-d692-45f8-84b9-150106ce03e9.png)

29. One feature to solve this problem is called BGP AS Override.  

**Cisco 8K Config Example** 

	ip as-path access-list 1 permit _398656$
	ip as-path access-list 1 permit _65515$
	ip as-path access-list 2 deny _398656$
	ip as-path access-list 2 permit .*
	ip as-path access-list 3 permit _398656$
	ip as-path access-list 3 permit _65515$
	ip as-path access-list 3 permit ^$
	ip as-path access-list 4 deny _398656$
	ip as-path access-list 4 deny _65515$
	ip as-path access-list 4 deny ^$
	ip as-path access-list 4 permit .*


	route-map HUB-ARS-IN permit 10 
 	match as-path 2
	!
	route-map AVS-TRANSIT-ARS-OUT permit 10 
 	set as-path replace any
	!
	route-map HUB-ARS-OUT permit 10 
 	set as-path replace 65515 12076 398656
 	set as-path replace 65515
 	set as-path replace 12076
	!
	route-map AVS-TRANSIT-ARS-IN permit 10 
 	match as-path 1


	router bgp <Your ASN>
 	bgp log-neighbor-changes
 	maximum-paths 2
 	aggregate-address <AVS /22 Management Address> 255.255.252.0 summary-only
 	neighbor <IPAddress of Hub vNET ARS Instance#1> remote-as 65515
 	neighbor <IPAddress of Hub vNET ARS Instance#1> description PEER-1 NET Hub vNET ARS
 	neighbor <IPAddress of Hub vNET ARS Instance#1> ebgp-multihop 255
 	neighbor <IPAddress of Hub vNET ARS Instance#1> soft-reconfiguration inbound
 	neighbor <IPAddress of Hub vNET ARS Instance#1> route-map HUB-ARS-IN in
 	neighbor <IPAddress of Hub vNET ARS Instance#1> route-map HUB-ARS-OUT out
 	neighbor <IPAddress of Hub vNET ARS Instance#1> filter-list 3 out

 	neighbor <IPAddress of Hub vNET ARS Instance#2> remote-as 65515
 	neighbor <IPAddress of Hub vNET ARS Instance#2> description PEER-2 Hub vNET ARS
 	neighbor <IPAddress of Hub vNET ARS Instance#2> ebgp-multihop 255
 	neighbor <IPAddress of Hub vNET ARS Instance#2> soft-reconfiguration inbound
 	neighbor <IPAddress of Hub vNET ARS Instance#2> route-map HUB-ARS-IN in
 	neighbor <IPAddress of Hub vNET ARS Instance#2> route-map HUB-ARS-OUT out
 	neighbor <IPAddress of Hub vNET ARS Instance#2> filter-list 3 out

 	neighbor <IPAddress of AVS Transit vNET ARS Instance#1> remote-as 65515
 	neighbor <IPAddress of AVS Transit vNET ARS Instance#1> description PEER-1 AVS vNET Transit ARS
 	neighbor <IPAddress of AVS Transit vNET ARS Instance#1> ebgp-multihop 255
 	neighbor <IPAddress of AVS Transit vNET ARS Instance#1> soft-reconfiguration inbound
 	neighbor <IPAddress of AVS Transit vNET ARS Instance#1> route-map AVS-TRANSIT-ARS-IN in
 	neighbor <IPAddress of AVS Transit vNET ARS Instance#1> route-map AVS-TRANSIT-ARS-OUT out
 	neighbor <IPAddress of AVS Transit vNET ARS Instance#1> filter-list 4 out

 	neighbor <IPAddress of AVS Transit vNET ARS Instance#2> remote-as 65515
 	neighbor <IPAddress of AVS Transit vNET ARS Instance#2> description PEER-2 AVS vNET Transit ARS
 	neighbor <IPAddress of AVS Transit vNET ARS Instance#2> ebgp-multihop 255
 	neighbor <IPAddress of AVS Transit vNET ARS Instance#2> soft-reconfiguration inbound
 	neighbor <IPAddress of AVS Transit vNET ARS Instance#2> route-map AVS-TRANSIT-ARS-IN in
 	neighbor <IPAddress of AVS Transit vNET ARS Instance#2> route-map AVS-TRANSIT-ARS-OUT out
 	neighbor <IPAddress of AVS Transit vNET ARS Instance#2> filter-list 4 out
 

	ip route 0.0.0.0 0.0.0.0 <Default Gateway of Outside Subnet>
	ip route <IPAddress of Hub vNET ARS Instance#1> 255.255.255.255 <Default Gateway of Outside Subnet>
	ip route <IPAddress of Hub vNET ARS Instance#2> 255.255.255.255 <Default Gateway of Outside Subnet>

	ip route <IPAddress of AVS Transit vNET ARS Instance#1> 255.255.255.255 <Default Gateway of Inside Subnet>
	ip route <IPAddress of AVS Transit vNET ARS Instance#2> 255.255.255.255 <Default Gateway of Inside Subnet>



