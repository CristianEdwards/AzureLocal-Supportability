# Azure Local private path Architecture Deep Dive

## Overview

If you want to use Azure Local without exposing your on-premises environment to the public internet, you can use the Arc gateway and the Azure Firewall Explicit Proxy feature to route all Azure Local traffic securely through your private connection (ExpressRoute or Site-to-Site VPN) to Azure.

This article explains the required components, the network flows and the steps to configure the Azure Local Private Path architecture.

### Key Benefits

- **Better Security:** Keep Azure Local traffic within your On-premises network and Azure using Express Route or S2S VPNs.
- **Easier Setup:** Leverage your existing network and security infrastructure while the Arc gateway manages Azure connectivity.
- **Simpler Management:** Fewer endpoints to manage means easier tracking and troubleshooting.

This guide explains how outbound connections work with the Arc gateway and Azure Local, including detailed diagrams and configuration requirements.

### Prerequisites and Network Requirements

- An existing Azure VNET without Azure Arc Private Link scope enabled. Other Private Link services such as Key Vault or Storage Private Links can be enabled on the VNET.
- An existing Azure ExpressRoute or Site-to-Site VPN connection from the on-premises environment to the Azure VNET.
- An Azure Firewall instance configured on the VNET without Azure Arc Private Link scope enabled.
- Network routing configured to allow Azure Local nodes to connect to the Azure VNET and subnets over ExpressRoute or S2S VPN.
- An Azure Arc gateway resource within the same subscription where Azure Local nodes are being registered.
- Azure Local version 2506 or later. Previous releases do not support private path architecture.

### Restrictions and limitations

- This solution uses Azure Firewall Explicit Proxy as a forward proxy. Azure Local does not support TLS inspection on the required endpoints.
- TLS certificates can't be applied to the Azure Firewall Explicit Proxy.

---

## Azure Local Private Path Required components

The following diagram introduces the core components involved in Azure Local private path:

- **Azure Local Instance:** Your on-premises Azure Local cluster.
- **Nodes:** Individual servers within your Azure Local instance.
- **Arc Proxy (on Arc connected machine agent):** A local proxy service running on each node, responsible for securely routing HTTPS traffic through the Arc gateway.
- **Arc Gateway Public Endpoint:** The Arc gateway instance must be created and configured on the same subscription where Azure Local nodes are registered. Once the Arc gateway resource is created, make sure the endpoint is allowed in your Azure Firewall policy application rule.
- **Enterprise Firewall:** Your organization's existing security infrastructure controlling outbound traffic.
- **Azure ExpressRoute:** The Express Route circuit is the communication path between the on-prem networks and Azure VNETs. Azure Local nodes traffic must be routed over express route to reach the Azure VNET. It is important to ensure that before Arc registration and enabling the Arc gateway, all nodes have network connectivity to the VNET and the subnets where Azure Firewall is running. All nodes must be able to reach the Azure Firewall private ip, as this will become the proxy endpoint for HTTP and HTTPS traffic from the nodes and the Arc proxy service.
- **Azure VNET:** Before deploying an Azure Firewall instance, an Azure VNET must be created with the corresponding subnets. At least one workloads subnet and one Azure Firewall subnet must be created. Azure Local does not support Azure Arc Private Link Scopes, and it is not supported to enable Azure Arc private link Scope on the VNET where Azure Firewall as explicit proxy is running. If Azure Arc for servers Private Link Scope is required for your workloads, a separate VNET with Azure Arc Private Link Scope configuration must be used.
- **Azure Firewall Explicit Proxy:** The Azure Firewall instance deployed on its own subnet should be configured as explicit proxy and the required Azure Local endpoints for Arc gateway scenarios should be allowed on the corresponding Azure Firewall policy Application Rules.
- **Azure Public Endpoints:** Azure services (e.g., Azure Resource Manager, Key Vault, Microsoft Graph) required by your local environment.

![Azure Local with Arc gateway outbound connectivity](./images/0-1NodePrivatePathComponents.dark.svg)

---
## Types of Network Traffic and Routing with Azure Local private path

When using Azure Local private path, operating system (OS) and Arc Resource Bridge appliance VM network traffic is categorized based on how it should be routed. Clearly distinguishing these categories helps administrators correctly configure network routing rules, ensuring secure, efficient, and compliant connectivity between on-premises infrastructure and Azure services over Azure ExpressRoute or Site to Site VPNs.

### Traffic Categories:

1. **🟦 OS HTTP and HTTPS traffic that must bypass Azure Firewall Explicit Proxy** 
   Specific HTTP and HTTPS connections that should not pass through Azure Firewall Explicit Proxy. Instead, these connections directly reach their intended internal destinations, typically due to technical requirements or performance considerations. Private Link endpoints such as Key Vault or Storage accounts should be also added to the proxy bypass list if your organization requires their usage.

2. **🟨 OS HTTP traffic that cannot use Arc proxy and must be sent to Azure Firewall Explicit Proxy**  
   HTTP traffic is incompatible with the Arc proxy. This traffic must instead be routed through Azure Firewall Explicit Proxy, ensuring compliance with internal security policies.

3. **🟩 OS HTTPS traffic that always uses Arc proxy**  
   HTTPS traffic that must always be routed through the Arc proxy. This ensures secure, controlled, and consistent connectivity to Azure endpoints, leveraging the Arc gateway's built-in security and management capabilities. Make sure you allow all the endpoints required for Arc gateway in Azure Local listed here: ![Azure Local required endpoints for Arc gateway](https://learn.microsoft.com/en-us/azure/azure-local/deploy/deployment-azure-arc-gateway-overview?view=azloc-2505&tabs=portal#azure-local-endpoints-not-redirected)

4. **🟥 Third-party OS HTTPS traffic not permitted through Arc gateway**
   All HTTPS traffic from the operating system initially goes to the Arc proxy. However, the Arc gateway only permits connections to Microsoft-managed endpoints. This means that HTTPS traffic destined for third-party services—such as OEM endpoints, hardware vendor update services, or other third-party agents installed on your servers, cannot pass through the Arc gateway. Instead, this traffic is redirected to Azure Firewall Explicit Proxy. To ensure these third-party services function correctly, you must explicitly configure Azure firewall Explicit Proxy Application Rules to allow access to these external endpoints based on your organization's requirements.

5. **📘 Arc Resource Bridge VM and AKS Clusters using Azure Local Instance cluster IP as proxy**
   Azure Arc Resource Bridge is a Kubernetes-based management solution deployed as a virtual appliance (also called the Arc appliance) on your on-premises infrastructure. Its main purpose is to enable your local resources to appear and be managed as Azure resources through Azure Resource Manager (ARM). To achieve this, the Arc Resource Bridge requires outbound connectivity to specific Azure endpoints. In an Azure Local environment, this outbound traffic is routed through the Cluster IP as proxy, which then securely forwards the traffic through the Arc gateway tunnel established by your Azure Local nodes. This approach simplifies network configuration, enhances security, and ensures compliance with your organization's network policies.
   Also, when deploying AKS cluster in Azure Local, by default the control plane VM and the pods will also use the Cluster IP as proxy to send the outbound traffic through the Arc gateway. However, for some services running inside your AKS clusters you might also need to allow additional endpoints that will be send directly to your firewall. 


## Traffic Flow Scenarios

### 1. Azure Local private path Node OS traffic bypassing Azure Firewall Explicit Proxy

This diagram illustrates traffic from Azure Local nodes that bypasses the Arc proxy entirely. Typical scenarios include:

- Internal communications within your local intranet.
- Node-to-node communications within the Azure Local cluster.
- Traffic destined for internal management or monitoring systems.
- Traffic destined for Private Link endpoints such as Key Vaults or Storage Accounts.

This traffic is sent directly to these endpoints without passing through the Arc gateway or Azure Firewall Explicit Proxy, ensuring low latency and efficient internal communication.

When defining your proxy bypass string for your Arc initialization script or when using the Companion App make sure your meet the following conditions:

- At least the IP address of each Azure Local machine.
- At least the IP address of the Cluster.
- At least the IPs you defined for your infrastructure network. Arc Resource Bridge, AKS, and future infrastructure services using these IPs require outbound connectivity.
- Or you can bypass the entire infrastructure subnet.
- NetBIOS name of each machine.
- NetBIOS name of the Cluster.
- Domain name or domain name with asterisk * wildcard at the beginning to include any host or subdomain.  For example, 192.168.1.* for subnets or *.contoso.com for domain names.
- Parameters must be separated with comma ,.
- CIDR notation to bypass subnets isn't supported.
- The use of <local> strings isn't supported in the proxy bypass list.

![Azure Local Node OS Traffic Bypassing Proxy](./images/1-1NodeIntranetBypassFlow.dark.svg)

---

### 2. Azure Local private path Node OS HTTP Traffic via Azure Firewall Explicit Proxy

This diagram shows how standard HTTP (non-HTTPS) traffic from Azure Local nodes is managed:

- HTTP traffic routes through Azure Firewall Explicit Proxy. Make sure you don't use a .local domain as your proxy server name. For example it is not supported to use proxy.local:8080 as proxy server. Use the proxy server IP instead if your proxy belongs to a .local domain.
- Make sure required endpoints for Azure Local are allowed in your enterprise firewall and in Azure Firewall Explicit Proxy, where your organization's security policies determine whether the traffic is allowed or blocked.

This ensures standard HTTP traffic aligns with your existing security infrastructure.

![Azure Local Node OS HTTP Traffic](./images/2-1NodeHTTPFlow.dark.svg)

---

### 3. Azure Local Node OS HTTPS Traffic via Arc Proxy

This diagram explains how HTTPS traffic from Azure Local nodes is securely routed:

- HTTPS traffic destined for allowed Azure endpoints routes through the Arc proxy running on each node. Make sure you allowed your Arc gateway URL in your proxy and/or firewall.
- The Arc proxy establishes a secure HTTPS tunnel to the Arc gateway public endpoint hosted in Azure.
- Traffic not allowed by the Arc proxy (non-approved endpoints) is redirected to Azure Firewall Explicit Proxy for further inspection or blocking. Make sure you allow the required 3rd party HTTPS endpoints for Azure Local such as the OEM SBE endpoints in Azure Firewall Explicit Proxy using the corresponding Application Rules.

This ensures secure, controlled, and compliant outbound HTTPS connectivity.

![Azure Local Node OS HTTPS Traffic](./images/3-1NodeHTTPSFlow.dark.svg)

---

### 4. Azure Resource Bridge Appliance VM HTTPS Traffic via Cluster IP Proxy

This diagram illustrates HTTPS traffic handling for the Azure Resource Bridge (ARB) appliance VM:

- ARB appliance VM sends HTTPS traffic through a Cluster IP proxy.
- The Cluster IP proxy securely routes allowed traffic through the Arc gateway's HTTPS tunnel to Azure.
- Non-allowed traffic is redirected to your firewall/proxy for security enforcement.

This ensures ARB appliance VM traffic is securely managed and compliant with your organization's policies.

![ARB Appliance VM HTTPS Traffic](./images/4-1NodeARBFlow.dark.svg)

---

### 5. AKS Clusters HTTPS Traffic via Cluster IP Proxy

This diagram shows HTTPS traffic handling for Azure Kubernetes Service (AKS) clusters within Azure Local:

- AKS clusters route HTTPS traffic through the Cluster IP proxy.
- The Cluster IP proxy securely forwards allowed traffic through the Arc gateway's HTTPS tunnel to Azure endpoints.
- Traffic not permitted by the Arc gateway is sent to your firewall/proxy for further security checks.

This ensures AKS clusters maintain secure and compliant outbound connectivity.

![AKS Clusters HTTPS Traffic](./images/5-1NodeAKSFlow.dark.svg)

---

### 6. Azure Local VMs HTTPS Traffic via Dedicated Arc Proxy

This diagram explains HTTPS traffic handling for Azure Local virtual machines (VMs):

- Each Azure Local VM uses its own dedicated Arc proxy to route HTTPS traffic.
- Allowed HTTPS traffic is securely tunneled through the Arc gateway to Azure public endpoints.
- Non-allowed traffic is redirected to Azure Firewall Explicit Proxy for security enforcement.

This ensures Azure Local VMs have secure, controlled, and compliant outbound connectivity.

![Azure Local VMs HTTPS Traffic](./images/6-1NodeArcVMFlow.dark.svg)

---

## How to configure Azure Local private path

### Step 1 - Create the Arc gateway resource in your subscription

The first step we must complete to enable the Azure Local private path Arc is to create the Arc gateway resource in our Azure subscription.
Create the Arc gateway resource in Azure
1.	Sign in to Azure portal.
2.	Go to the Azure Arc > Azure Arc gateway page, then select Create.
3.	Select the subscription and resource group where you want the Arc gateway resource to be managed within Azure. An Arc gateway resource can be used by any Arc-enabled resource in the same Azure tenant.
4.	For Name, enter the name for the Arc gateway resource.
5.	For Location, enter the region where the Arc gateway resource should live. An Arc gateway resource can be used by any Arc-enabled resource in the same Azure tenant.
6.	Select Next.
7.	On the Tags page, specify one or more custom tags to support your standards.
8.	Select Review & Create.
9.	Review your details, and then select Create.
The gateway creation process takes nine to ten minutes to complete.
Once the Arc gateway is created, the URL gets assigned. Take note of this URL because you will need to allow HTTPS traffic to this endpoint in your Azure Firewall application rules.

![Arc gateway URL](./images/Arcgatewaycreation.png)

You can also create an Arc gateway resource using Azure CLI, or Azure PowerShell. Please refer to the the Azure Local Arc gateway documentation if using the Azure Portal to create the Arc gateway is not an option. Overview of Azure Arc gateway for Azure Local, version 23H2 (preview) - Azure Local | Microsoft Learn

---
## Summary of the Overall Connectivity Model

- **Allowed HTTPS traffic** is securely tunneled through the Arc gateway, significantly reducing firewall rules required (fewer than 30 endpoints).
- **Non-allowed traffic** (highlighted in pink in diagrams) is redirected to Azure Firewall Explicit Proxy for inspection and enforcement.
- **Internal and Private Link traffic** bypasses proxies entirely, ensuring efficient local communication.

This structured approach simplifies network management, enhances security, and ensures compliance with organizational policies.