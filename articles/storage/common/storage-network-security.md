---
title: Configure Azure Storage firewalls and virtual networks
description: Configure layered network security for your storage account by using Azure Storage firewalls and Azure Virtual Network.
services: storage
author: jimmart-dev
ms.service: storage
ms.topic: how-to
ms.date: 04/20/2023
ms.author: jammart
ms.reviewer: santoshc
ms.subservice: common 
ms.custom: devx-track-azurepowershell, devx-track-azurecli, build-2023
---

# Configure Azure Storage firewalls and virtual networks

Azure Storage provides a layered security model. This model enables you to control the level of access to your storage accounts that your applications and enterprise environments demand, based on the type and subset of networks or resources that you use.

When you configure network rules, only applications that request data over the specified set of networks or through the specified set of Azure resources can access a storage account. You can limit access to your storage account to requests that come from specified IP addresses, IP ranges, subnets in an Azure virtual network, or resource instances of some Azure services.

Storage accounts have a public endpoint that's accessible through the internet. You can also create [private endpoints for your storage account](storage-private-endpoints.md). Creating private endpoints assigns a private IP address from your virtual network to the storage account. It helps secure traffic between your virtual network and the storage account over a private link.

The Azure Storage firewall provides access control for the public endpoint of your storage account. You can also use the firewall to block all access through the public endpoint when you're using private endpoints. Your firewall configuration also enables trusted Azure platform services to access the storage account.

An application that accesses a storage account when network rules are in effect still requires proper authorization for the request. Authorization is supported with Azure Active Directory (Azure AD) credentials for blobs and queues, with a valid account access key, or with a shared access signature (SAS) token. When you configure a blob container for anonymous public access, requests to read data in that container don't need to be authorized. The firewall rules remain in effect and will block anonymous traffic.

Turning on firewall rules for your storage account blocks incoming requests for data by default, unless the requests originate from a service that operates within an Azure virtual network or from allowed public IP addresses. Requests that are blocked include those from other Azure services, from the Azure portal, and from logging and metrics services.

You can grant access to Azure services that operate from within a virtual network by allowing traffic from the subnet that hosts the service instance. You can also enable a limited number of scenarios through the exceptions mechanism that this article describes. To access data from the storage account through the Azure portal, you need to be on a machine within the trusted boundary (either IP or virtual network) that you set up.

[!INCLUDE [updated-for-az](../../../includes/updated-for-az.md)]

## Scenarios

To secure your storage account, you should first configure a rule to deny access to traffic from all networks (including internet traffic) on the public endpoint, by default. Then, you should configure rules that grant access to traffic from specific virtual networks. You can also configure rules to grant access to traffic from selected public internet IP address ranges, enabling connections from specific internet or on-premises clients. This configuration helps you build a secure network boundary for your applications.

You can combine firewall rules that allow access from specific virtual networks and from public IP address ranges on the same storage account. You can apply storage firewall rules to existing storage accounts or when you create new storage accounts.

Storage firewall rules apply to the public endpoint of a storage account. You don't need any firewall access rules to allow traffic for private endpoints of a storage account. The process of approving the creation of a private endpoint grants implicit access to traffic from the subnet that hosts the private endpoint.

Network rules are enforced on all network protocols for Azure Storage, including REST and SMB. To access data by using tools such as the Azure portal, Azure Storage Explorer, and AzCopy, you must configure explicit network rules.

After you apply network rules, they're enforced for all requests. SAS tokens that grant access to a specific IP address serve to limit the access of the token holder, but they don't grant new access beyond configured network rules.

Network rules don't affect virtual machine (VM) disk traffic, including mount and unmount operations and disk I/O. Network rules help protect REST access to page blobs.

Classic storage accounts don't support firewalls and virtual networks.

You can use unmanaged disks in storage accounts with network rules applied to back up and restore VMs by creating an exception. The [Manage exceptions](#manage-exceptions) section of this article documents this process. Firewall exceptions aren't applicable with managed disks, because Azure already manages them.

## Change the default network access rule

By default, storage accounts accept connections from clients on any network. You can limit access to selected networks *or* prevent traffic from all networks and permit access only through a [private endpoint](storage-private-endpoints.md).

You must set the default rule to **deny**, or network rules have no effect. However, changing this setting can affect your application's ability to connect to Azure Storage. Be sure to grant access to any allowed networks or set up access through a private endpoint before you change this setting.

### [Portal](#tab/azure-portal)

1. Go to the storage account that you want to secure.

2. Locate the **Networking** settings under **Security + networking**.

3. Choose which type of public network access you want to allow:

   - To allow traffic from all networks, select **Enabled from all networks**.

   - To allow traffic only from specific virtual networks, select **Enabled from selected virtual networks and IP addresses**.

   - To block traffic from all networks, select **Disabled**.

4. Select **Save** to apply your changes.

<a id="powershell"></a>

### [PowerShell](#tab/azure-powershell)

1. Install [Azure PowerShell](/powershell/azure/install-azure-powershell) and [sign in](/powershell/azure/authenticate-azureps).

2. Choose which type of public network access you want to allow:

    - To allow traffic from all networks, use the `Update-AzStorageAccountNetworkRuleSet` command and set the `-DefaultAction` parameter to `Allow`:

      ```powershell
      Update-AzStorageAccountNetworkRuleSet -ResourceGroupName "myresourcegroup" -Name "mystorageaccount" -DefaultAction Allow
      ```
  
    - To allow traffic only from specific virtual networks, use the `Update-AzStorageAccountNetworkRuleSet` command and set the `-DefaultAction` parameter to `Deny`:

      ```powershell
      Update-AzStorageAccountNetworkRuleSet -ResourceGroupName "myresourcegroup" -Name "mystorageaccount" -DefaultAction Deny
      ```

    - To block traffic from all networks, use the `Set-AzStorageAccount` command and set the `-PublicNetworkAccess` parameter to `Disabled`. Traffic will be allowed only through a [private endpoint](storage-private-endpoints.md). You'll have to create that private endpoint.

      ```powershell
      Set-AzStorageAccount -ResourceGroupName "myresourcegroup" -Name "mystorageaccount" -PublicNetworkAccess Disabled
      ```

### [Azure CLI](#tab/azure-cli)

1. Install the [Azure CLI](/cli/azure/install-azure-cli) and [sign in](/cli/azure/authenticate-azure-cli).

2. Choose which type of public network access you want to allow:

    - To allow traffic from all networks, use the `az storage account update` command and set the `--default-action` parameter to `Allow`:

      ```azurecli
      az storage account update --resource-group "myresourcegroup" --name "mystorageaccount" --default-action Allow
      ```

    - To allow traffic only from specific virtual networks, use the `az storage account update` command and set the `--default-action` parameter to `Deny`:

      ```azurecli
      az storage account update --resource-group "myresourcegroup" --name "mystorageaccount" --default-action Deny
      ```

    - To block traffic from all networks, use the `az storage account update` command and set the `--public-network-access` parameter to `Disabled`. Traffic will be allowed only through a [private endpoint](storage-private-endpoints.md). You'll have to create that private endpoint.

      ```azurecli
      az storage account update --name MyStorageAccount --resource-group MyResourceGroup --public-network-access Disabled
      ```

---

> [!CAUTION]
> By design, access to a storage account from trusted services takes the highest precedence over other network access restrictions. If you set **Public network access** to **Disabled** after previously setting it to **Enabled from selected virtual networks and IP addresses**, any [resource instances](#grant-access-from-azure-resource-instances) and [exceptions](#manage-exceptions) that you previously configured, including [Allow Azure services on the trusted services list to access this storage account](#grant-access-to-trusted-azure-services), will remain in effect. As a result, those resources and services might still have access to the storage account.

## Grant access from a virtual network

You can configure storage accounts to allow access only from specific subnets. The allowed subnets can belong to a virtual network in the same subscription or a different subscription, including those that belong to a different Azure AD tenant. With [cross-region service endpoints](#azure-storage-cross-region-service-endpoints), the allowed subnets can also be in different regions from the storage account.

You can enable a [service endpoint](../../virtual-network/virtual-network-service-endpoints-overview.md) for Azure Storage within the virtual network. The service endpoint routes traffic from the virtual network through an optimal path to the Azure Storage service. The identities of the subnet and the virtual network are also transmitted with each request. Administrators can then configure network rules for the storage account that allow requests to be received from specific subnets in a virtual network. Clients granted access via these network rules must continue to meet the authorization requirements of the storage account to access the data.

Each storage account supports up to 200 virtual network rules. You can combine these rules with [IP network rules](#grant-access-from-an-internet-ip-range).

> [!IMPORTANT]
> If you delete a subnet that's included in a network rule, it will be removed from the network rules for the storage account. If you create a new subnet by the same name, it won't have access to the storage account. To allow access, you must explicitly authorize the new subnet in the network rules for the storage account.

### Required permissions

To apply a virtual network rule to a storage account, the user must have the appropriate permissions for the subnets that are being added. A [Storage Account Contributor](../../role-based-access-control/built-in-roles.md#storage-account-contributor) or a user who has permission to the `Microsoft.Network/virtualNetworks/subnets/joinViaServiceEndpoint/action` [Azure resource provider operation](../../role-based-access-control/resource-provider-operations.md#microsoftnetwork) can apply a rule by using a custom Azure role.

The storage account and the virtual networks that get access can be in different subscriptions, including subscriptions that are a part of a different Azure AD tenant.

Configuration of rules that grant access to subnets in virtual networks that are a part of a different Azure AD tenant are currently supported only through PowerShell, the Azure CLI, and REST APIs. You can't configure such rules through the Azure portal, though you can view them in the portal.

### Azure Storage cross-region service endpoints

Cross-region service endpoints for Azure Storage became generally available in April 2023. They work between virtual networks and storage service instances in any region. With cross-region service endpoints, subnets no longer use a public IP address to communicate with any storage account, including those in another region. Instead, all the traffic from subnets to storage accounts uses a private IP address as a source IP. As a result, any storage accounts that use IP network rules to permit traffic from those subnets no longer have an effect.

Configuring service endpoints between virtual networks and service instances in a [paired region](../../best-practices-availability-paired-regions.md) can be an important part of your disaster recovery plan. Service endpoints allow continuity during a regional failover and access to read-only geo-redundant storage (RA-GRS) instances. Network rules that grant access from a virtual network to a storage account also grant access to any RA-GRS instance.

When you're planning for disaster recovery during a regional outage, you should create the virtual networks in the paired region in advance. Enable service endpoints for Azure Storage, with network rules granting access from these alternative virtual networks. Then apply these rules to your geo-redundant storage accounts.

Local and cross-region service endpoints can't coexist on the same subnet. To replace existing service endpoints with cross-region ones, delete the existing `Microsoft.Storage` endpoints and re-create them as cross-region endpoints (`Microsoft.Storage.Global`).

### Managing virtual network rules

You can manage virtual network rules for storage accounts through the Azure portal, PowerShell, or the Azure CLI v2.

If you want to enable access to your storage account from a virtual network or subnet in another Azure AD tenant, you must use PowerShell or the Azure CLI. The Azure portal does not show subnets in other Azure AD tenants.

#### [Portal](#tab/azure-portal)

1. Go to the storage account that you want to secure.

2. Select **Networking**.

3. Check that you've chosen to allow access from **Selected networks**.

4. To grant access to a virtual network by using a new network rule, under **Virtual networks**, select **Add existing virtual network**. Select the **Virtual networks** and **Subnets** options, and then select **Add**.

    To create a new virtual network and grant it access, select **Add new virtual network**. Provide the necessary information to create the new virtual network, and then select **Create**.

    If a service endpoint for Azure Storage wasn't previously configured for the selected virtual network and subnets, you can configure it as part of this operation.

    Presently, only virtual networks that belong to the same Azure AD tenant appear for selection during rule creation. To grant access to a subnet in a virtual network that belongs to another tenant, use PowerShell, the Azure CLI, or REST APIs.

5. To remove a virtual network or subnet rule, select the ellipsis (**...**) to open the context menu for the virtual network or subnet, and then select **Remove**.

6. Select **Save** to apply your changes.

#### [PowerShell](#tab/azure-powershell)

1. Install [Azure PowerShell](/powershell/azure/install-azure-powershell) and [sign in](/powershell/azure/authenticate-azureps).

2. List virtual network rules:

    ```powershell
    (Get-AzStorageAccountNetworkRuleSet -ResourceGroupName "myresourcegroup" -AccountName "mystorageaccount").VirtualNetworkRules
    ```

3. Enable a service endpoint for Azure Storage on an existing virtual network and subnet:

    ```powershell
    Get-AzVirtualNetwork -ResourceGroupName "myresourcegroup" -Name "myvnet" | Set-AzVirtualNetworkSubnetConfig -Name "mysubnet" -AddressPrefix "10.0.0.0/24" -ServiceEndpoint "Microsoft.Storage.Global" | Set-AzVirtualNetwork
    ```

4. Add a network rule for a virtual network and subnet:

    ```powershell
    $subnet = Get-AzVirtualNetwork -ResourceGroupName "myresourcegroup" -Name "myvnet" | Get-AzVirtualNetworkSubnetConfig -Name "mysubnet"
    Add-AzStorageAccountNetworkRule -ResourceGroupName "myresourcegroup" -Name "mystorageaccount" -VirtualNetworkResourceId $subnet.Id
    ```

    To add a network rule for a subnet in a virtual network that belongs to another Azure AD tenant, use a fully qualified `VirtualNetworkResourceId` parameter in the form `/subscriptions/subscription-ID/resourceGroups/resourceGroup-Name/providers/Microsoft.Network/virtualNetworks/vNet-name/subnets/subnet-name`.

5. Remove a network rule for a virtual network and subnet:

    ```powershell
    $subnet = Get-AzVirtualNetwork -ResourceGroupName "myresourcegroup" -Name "myvnet" | Get-AzVirtualNetworkSubnetConfig -Name "mysubnet"
    Remove-AzStorageAccountNetworkRule -ResourceGroupName "myresourcegroup" -Name "mystorageaccount" -VirtualNetworkResourceId $subnet.Id
    ```

#### [Azure CLI](#tab/azure-cli)

1. Install the [Azure CLI](/cli/azure/install-azure-cli) and [sign in](/cli/azure/authenticate-azure-cli).

2. List virtual network rules:

    ```azurecli
    az storage account network-rule list --resource-group "myresourcegroup" --account-name "mystorageaccount" --query virtualNetworkRules
    ```

3. Enable a service endpoint for Azure Storage on an existing virtual network and subnet:

    ```azurecli
    az network vnet subnet update --resource-group "myresourcegroup" --vnet-name "myvnet" --name "mysubnet" --service-endpoints "Microsoft.Storage.Global"
    ```

4. Add a network rule for a virtual network and subnet:

    ```azurecli
    subnetid=$(az network vnet subnet show --resource-group "myresourcegroup" --vnet-name "myvnet" --name "mysubnet" --query id --output tsv)
    az storage account network-rule add --resource-group "myresourcegroup" --account-name "mystorageaccount" --subnet $subnetid
    ```

    To add a rule for a subnet in a virtual network that belongs to another Azure AD tenant, use a fully qualified subnet ID in the form `/subscriptions/<subscription-ID>/resourceGroups/<resourceGroup-Name>/providers/Microsoft.Network/virtualNetworks/<vNet-name>/subnets/<subnet-name>`. You can use the `subscription` parameter to retrieve the subnet ID for a virtual network that belongs to another Azure AD tenant.

5. Remove a network rule for a virtual network and subnet:

    ```azurecli
    subnetid=$(az network vnet subnet show --resource-group "myresourcegroup" --vnet-name "myvnet" --name "mysubnet" --query id --output tsv)
    az storage account network-rule remove --resource-group "myresourcegroup" --account-name "mystorageaccount" --subnet $subnetid
    ```

---

## Grant access from an internet IP range

You can use IP network rules to allow access from specific public internet IP address ranges by creating IP network rules. Each storage account supports up to 200 rules. These rules grant access to specific internet-based services and on-premises networks and block general internet traffic.

The following restrictions apply to IP address ranges:

- IP network rules are allowed only for *public internet* IP addresses.

  IP address ranges reserved for private networks (as defined in [RFC 1918](https://tools.ietf.org/html/rfc1918#section-3)) aren't allowed in IP rules. Private networks include addresses that start with 10, 172.16 to 172.31, and 192.168.

- You must provide allowed internet address ranges by using [CIDR notation](https://tools.ietf.org/html/rfc4632) in the form 16.17.18.0/24 or as individual IP addresses like 16.17.18.19.

- Small address ranges that use /31 or /32 prefix sizes are not supported. Configure these ranges by using individual IP address rules.

- Only IPv4 addresses are supported for configuration of storage firewall rules.

You can't use IP network rules in the following cases:

- To restrict access to clients in same Azure region as the storage account.

  IP network rules have no effect on requests that originate from the same Azure region as the storage account. Use [Virtual network rules](#grant-access-from-a-virtual-network) to allow same-region requests.

- To restrict access to clients in a [paired region](../../availability-zones/cross-region-replication-azure.md) that are in a virtual network that has a service endpoint.

- To restrict access to Azure services deployed in the same region as the storage account.

  Services deployed in the same region as the storage account use private Azure IP addresses for communication. So, you can't restrict access to specific Azure services based on their public outbound IP address range.

### Configuring access from on-premises networks

To grant access from your on-premises networks to your storage account by using an IP network rule, you must identify the internet-facing IP addresses that your network uses. Contact your network administrator for help.

If you're using [Azure ExpressRoute](../../expressroute/expressroute-introduction.md) from your premises, for public peering or Microsoft peering, you need to identify the NAT IP addresses that are used. For public peering, each ExpressRoute circuit (by default) uses two NAT IP addresses applied to Azure service traffic when the traffic enters the Microsoft Azure network backbone. For Microsoft peering, either the service provider or the customer provides the NAT IP addresses.

To allow access to your service resources, you must allow these public IP addresses in the firewall setting for resource IPs. To find your IP addresses for public-peering ExpressRoute circuits, [open a support ticket with ExpressRoute](https://portal.azure.com/#blade/Microsoft_Azure_Support/HelpAndSupportBlade/overview) via the Azure portal. [Learn more about NAT for ExpressRoute public peering and Microsoft peering](../../expressroute/expressroute-nat.md#nat-requirements-for-azure-public-peering).

### Managing IP network rules

You can manage IP network rules for storage accounts through the Azure portal, PowerShell, or the Azure CLI v2.

#### [Portal](#tab/azure-portal)

1. Go to the storage account that you want to secure.

2. Select **Networking**.

3. Check that you've chosen to allow access from **Selected networks**.

4. To grant access to an internet IP range, enter the IP address or address range (in CIDR format) under **Firewall** > **Address Range**.

5. To remove an IP network rule, select the delete icon (:::image type="icon" source="media/storage-network-security/delete-icon.png":::) next to the address range.

6. Select **Save** to apply your changes.

#### [PowerShell](#tab/azure-powershell)

1. Install [Azure PowerShell](/powershell/azure/install-azure-powershell) and [sign in](/powershell/azure/authenticate-azureps).

2. List IP network rules:

    ```powershell
    (Get-AzStorageAccountNetworkRuleSet -ResourceGroupName "myresourcegroup" -AccountName "mystorageaccount").IPRules
    ```

3. Add a network rule for an individual IP address:

    ```powershell
    Add-AzStorageAccountNetworkRule -ResourceGroupName "myresourcegroup" -AccountName "mystorageaccount" -IPAddressOrRange "16.17.18.19"
    ```

4. Add a network rule for an IP address range:

    ```powershell
    Add-AzStorageAccountNetworkRule -ResourceGroupName "myresourcegroup" -AccountName "mystorageaccount" -IPAddressOrRange "16.17.18.0/24"
    ```

5. Remove a network rule for an individual IP address:

    ```powershell
    Remove-AzStorageAccountNetworkRule -ResourceGroupName "myresourcegroup" -AccountName "mystorageaccount" -IPAddressOrRange "16.17.18.19"
    ```

6. Remove a network rule for an IP address range:

    ```powershell
    Remove-AzStorageAccountNetworkRule -ResourceGroupName "myresourcegroup" -AccountName "mystorageaccount" -IPAddressOrRange "16.17.18.0/24"
    ```

#### [Azure CLI](#tab/azure-cli)

1. Install the [Azure CLI](/cli/azure/install-azure-cli) and [sign in](/cli/azure/authenticate-azure-cli).

1. List IP network rules:

    ```azurecli
    az storage account network-rule list --resource-group "myresourcegroup" --account-name "mystorageaccount" --query ipRules
    ```

1. Add a network rule for an individual IP address:

    ```azurecli
    az storage account network-rule add --resource-group "myresourcegroup" --account-name "mystorageaccount" --ip-address "16.17.18.19"
    ```

1. Add a network rule for an IP address range:

    ```azurecli
    az storage account network-rule add --resource-group "myresourcegroup" --account-name "mystorageaccount" --ip-address "16.17.18.0/24"
    ```

1. Remove a network rule for an individual IP address:

    ```azurecli
    az storage account network-rule remove --resource-group "myresourcegroup" --account-name "mystorageaccount" --ip-address "16.17.18.19"
    ```

1. Remove a network rule for an IP address range:

    ```azurecli
    az storage account network-rule remove --resource-group "myresourcegroup" --account-name "mystorageaccount" --ip-address "16.17.18.0/24"
    ```

---

<a id="grant-access-specific-instances"></a>

## Grant access from Azure resource instances

In some cases, an application might depend on Azure resources that can't be isolated through a virtual network or an IP address rule. But you still want to secure and restrict storage account access to only your application's Azure resources. You can configure storage accounts to allow access to specific resource instances of some Azure services by creating a resource instance rule.

The Azure role assignments of the resource instance determine the types of operations that a resource instance can perform on storage account data. Resource instances must be from the same tenant as your storage account, but they can belong to any subscription in the tenant.

### [Portal](#tab/azure-portal)

You can add or remove resource network rules in the Azure portal:

1. Sign in to the [Azure portal](https://portal.azure.com/).

2. Locate your storage account and display the account overview.

3. Select **Networking**.

4. Under **Firewalls and virtual networks**, for **Selected networks**, select the option to allow access.

5. Scroll down to find **Resource instances**. In the **Resource type** dropdown list, select the resource type of your resource instance.

6. In the **Instance name** dropdown list, select the resource instance. You can also choose to include all resource instances in the active tenant, subscription, or resource group.

7. Select **Save** to apply your changes. The resource instance appears in the **Resource instances** section of the page for network settings.

To remove the resource instance, select the delete icon (:::image type="icon" source="media/storage-network-security/delete-icon.png":::) next to the resource instance.

### [PowerShell](#tab/azure-powershell)

You can use PowerShell commands to add or remove resource network rules.

#### Grant access

Add a network rule that grants access from a resource instance:

```powershell
$resourceId = "/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/myResourceGroup/providers/Microsoft.DataFactory/factories/myDataFactory"
$tenantId = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
$resourceGroupName = "myResourceGroup"
$accountName = "mystorageaccount"

Add-AzStorageAccountNetworkRule -ResourceGroupName $resourceGroupName -Name $accountName -TenantId $tenantId -ResourceId $resourceId

```

Specify multiple resource instances at once by modifying the network rule set:

```powershell
$resourceId1 = "/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/myResourceGroup/providers/Microsoft.DataFactory/factories/myDataFactory"
$resourceId2 = "/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/myResourceGroup/providers/Microsoft.Sql/servers/mySQLServer"
$tenantId = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
$resourceGroupName = "myResourceGroup"
$accountName = "mystorageaccount"

Update-AzStorageAccountNetworkRuleSet -ResourceGroupName $resourceGroupName -Name $accountName -ResourceAccessRule (@{ResourceId=$resourceId1;TenantId=$tenantId},@{ResourceId=$resourceId2;TenantId=$tenantId}) 
```

#### Remove access

Remove a network rule that grants access from a resource instance:

```powershell
$resourceId = "/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/myResourceGroup/providers/Microsoft.DataFactory/factories/myDataFactory"
$tenantId = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
$resourceGroupName = "myResourceGroup"
$accountName = "mystorageaccount"

Remove-AzStorageAccountNetworkRule -ResourceGroupName $resourceGroupName -Name $accountName -TenantId $tenantId -ResourceId $resourceId  
```

Remove all network rules that grant access from resource instances:

```powershell
$resourceGroupName = "myResourceGroup"
$accountName = "mystorageaccount"

Update-AzStorageAccountNetworkRuleSet -ResourceGroupName $resourceGroupName -Name $accountName -ResourceAccessRule @()  
```

#### View a list of allowed resource instances

View a complete list of resource instances that have access to the storage account:

```powershell
$resourceGroupName = "myResourceGroup"
$accountName = "mystorageaccount"

$rule = Get-AzStorageAccountNetworkRuleSet -ResourceGroupName $resourceGroupName -Name $accountName
$rule.ResourceAccessRules 
```

### [Azure CLI](#tab/azure-cli)

You can use Azure CLI commands to add or remove resource network rules.

#### Grant access

Add a network rule that grants access from a resource instance:

```azurecli
az storage account network-rule add \
    --resource-id /subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/myResourceGroup/providers/Microsoft.Synapse/workspaces/testworkspace \
    --tenant-id xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx \
    -g myResourceGroup \
    --account-name mystorageaccount
```

#### Remove access

Remove a network rule that grants access from a resource instance:

```azurecli
az storage account network-rule remove \
    --resource-id /subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/myResourceGroup/providers/Microsoft.Synapse/workspaces/testworkspace \
    --tenant-id xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx \
    -g myResourceGroup \
    --account-name mystorageaccount
```

#### View a list of allowed resource instances

View a complete list of resource instances that have access to the storage account:

```azurecli
az storage account network-rule list \
    -g myResourceGroup \
    --account-name mystorageaccount
```

---

<a id="exceptions"></a>
<a id="trusted-microsoft-services"></a>

## Grant access to trusted Azure services

Some Azure services operate from networks that you can't include in your network rules. You can grant a subset of such trusted Azure services access to the storage account, while maintaining network rules for other apps. These trusted services will then use strong authentication to connect to your storage account.

You can grant access to trusted Azure services by creating a network rule exception. The [Manage exceptions](#manage-exceptions) section of this article provides step-by-step guidance.

<a id="trusted-access-resources-in-subscription"></a>

### Trusted access for resources registered in your subscription

Resources of some services that are registered in your subscription can access your storage account *in the same subscription* for selected operations, such as writing logs or running backups.  The following table describes each service and the allowed operations.

| Service                  | Resource provider name     | Allowed operations                 |
|:------------------------ |:-------------------------- |:---------------------------------- |
| Azure Backup             | `Microsoft.RecoveryServices` | Run backups and restores of unmanaged disks in infrastructure as a service (IaaS) virtual machines (not required for managed disks). [Learn more](../../backup/backup-overview.md). |
| Azure Data Box           | `Microsoft.DataBox`          | Import data to Azure. [Learn more](../../databox/data-box-overview.md). |
| Azure DevTest Labs       | `Microsoft.DevTestLab`       | Create custom images and install artifacts. [Learn more](../../devtest-labs/devtest-lab-overview.md). |
| Azure Event Grid         | `Microsoft.EventGrid`        | Enable [Azure Blob Storage event publishing](../../event-grid/concepts.md#event-sources) and allow [publishing to storage queues](../../event-grid/event-handlers.md). |
| Azure Event Hubs         | `Microsoft.EventHub`         | Archive data by using Event Hubs Capture. [Learn More](../../event-hubs/event-hubs-capture-overview.md). |
| Azure File Sync          | `Microsoft.StorageSync`      | Transform your on-premises file server to a cache for Azure file shares. This capability allows multiple-site sync, fast disaster recovery, and cloud-side backup. [Learn more](../file-sync/file-sync-planning.md). |
| Azure HDInsight          | `Microsoft.HDInsight`        | Provision the initial contents of the default file system for a new HDInsight cluster. [Learn more](../../hdinsight/hdinsight-hadoop-use-blob-storage.md). |
| Azure Import/Export      | `Microsoft.ImportExport`     | Import data to Azure Storage or export data from Azure Storage. [Learn more](../../import-export/storage-import-export-service.md).  |
| Azure Monitor            | `Microsoft.Insights`         | Write monitoring data to a secured storage account, including resource logs, Azure AD sign-in and audit logs, and Microsoft Intune logs. [Learn more](../../azure-monitor/roles-permissions-security.md). |
| Azure networking services         | `Microsoft.Network`          | Store and analyze network traffic logs, including through the Azure Network Watcher and Azure Traffic Manager services. [Learn more](../../network-watcher/network-watcher-nsg-flow-logging-overview.md). |
| Azure Site Recovery      | `Microsoft.SiteRecovery`     | Enable replication for disaster recovery of Azure IaaS virtual machines when you're using firewall-enabled cache, source, or target storage accounts.  [Learn more](../../site-recovery/azure-to-azure-tutorial-enable-replication.md). |

<a id="trusted-access-system-assigned-managed-identity"></a>
<a id="trusted-access-based-on-system-assigned-managed-identity"></a>

### Trusted access based on a managed identity

The following table lists services that can access your storage account data if the resource instances of those services have the appropriate permission.

| Service                        | Resource provider name                 | Purpose            |
| :----------------------------- | :------------------------------------- | :----------------- |
| Azure API Management           | `Microsoft.ApiManagement/service`        | Enables access to storage accounts behind firewalls via policies. [Learn more](../../api-management/authentication-managed-identity-policy.md#use-managed-identity-in-send-request-policy). |
| Azure Cache for Redis | `Microsoft.Cache/Redis` | Enables access to storage accounts. [Learn more](../../azure-cache-for-redis/cache-managed-identity.md).| 
| Azure Cognitive Search         | `Microsoft.Search/searchServices`        | Enables access to storage accounts for indexing, processing, and querying. |
| Azure Cognitive Services       | `Microsoft.CognitiveService/accounts`    | Enables access to storage accounts. [Learn more](../..//cognitive-services/cognitive-services-virtual-networks.md).|
| Azure Container Registry | `Microsoft.ContainerRegistry/registries` | Through the ACR Tasks suite of features, enables access to storage accounts when you're building container images. |
| Azure Data Factory             | `Microsoft.DataFactory/factories`        | Enables access to storage accounts through the Data Factory runtime. |
| Azure Data Share               | `Microsoft.DataShare/accounts`           | Enables access to storage accounts. |
| Azure DevTest Labs             | `Microsoft.DevTestLab/labs`              | Enables access to storage accounts. |
| Azure Event Grid  | `Microsoft.EventGrid/topics` | Enables access to storage accounts. |
| Azure Healthcare APIs | `Microsoft.HealthcareApis/services` | Enables access to storage accounts. |
| Azure IoT Central | `Microsoft.IoTCentral/IoTApps` | Enables access to storage accounts. |
| Azure IoT Hub                  | `Microsoft.Devices/IotHubs`              | Allows data from an IoT hub to be written to Blob Storage. [Learn more](../../iot-hub/virtual-network-support.md#egress-connectivity-from-iot-hub-to-other-azure-resources). |
| Azure Logic Apps               | `Microsoft.Logic/workflows`              | Enables logic apps to access storage accounts. [Learn more](../../logic-apps/create-managed-service-identity.md#authenticate-access-with-managed-identity). |
| Azure Machine Learning | `Microsoft.MachineLearningServices`      | Enables authorized Azure Machine Learning workspaces to write experiment output, models, and logs to Blob Storage and read the data. [Learn more](../../machine-learning/how-to-network-security-overview.md#secure-the-workspace-and-associated-resources). |
| Azure Media Services           | `Microsoft.Media/mediaservices`          | Enables access to storage accounts. |
| Azure Migrate                  | `Microsoft.Migrate/migrateprojects`      | Enables access to storage accounts. |
| Microsoft Purview                  | `Microsoft.Purview/accounts`             | Enables access to storage accounts. |
| Azure Site Recovery            | `Microsoft.RecoveryServices/vaults`      | Enables access to storage accounts. |
| Azure SQL Database             | `Microsoft.Sql`                          | Allows [writing audit data to storage accounts behind a firewall](/azure/azure-sql/database/audit-write-storage-account-behind-vnet-firewall). |
| Azure Synapse Analytics        | `Microsoft.Sql`                          | Allows import and export of data from specific SQL databases via the `COPY` statement or PolyBase (in a dedicated pool), or the `openrowset` function and external tables in a serverless pool. [Learn more](/azure/azure-sql/database/vnet-service-endpoint-rule-overview). |
| Azure Stream Analytics         | `Microsoft.StreamAnalytics`              | Allows data from a streaming job to be written to Blob Storage. [Learn more](../../stream-analytics/blob-output-managed-identity.md). |
| Azure Synapse Analytics        | `Microsoft.Synapse/workspaces`           | Enables access to data in Azure Storage. |

If your account doesn't have the hierarchical namespace feature enabled on it, you can grant permission by explicitly assigning an Azure role to the [managed identity](../../active-directory/managed-identities-azure-resources/overview.md) for each resource instance. In this case, the scope of access for the instance corresponds to the Azure role that's assigned to the managed identity.

You can use the same technique for an account that has the hierarchical namespace feature enabled on it. However, you don't have to assign an Azure role if you add the managed identity to the access control list (ACL) of any directory or blob that the storage account contains. In that case, the scope of access for the instance corresponds to the directory or file to which the managed identity has access.

You can also combine Azure roles and ACLs together to grant access. To learn more, see [Access control model in Azure Data Lake Storage Gen2](../blobs/data-lake-storage-access-control-model.md).

We recommend that you [use resource instance rules to grant access to specific resources](#grant-access-from-azure-resource-instances).

<a id="manage-exceptions"></a>

## Manage exceptions

In some cases, like storage analytics, access to read resource logs and metrics is required from outside the network boundary. When you configure trusted services to access the storage account, you can allow read access for the log files, metrics tables, or both by creating a network rule exception. You can manage network rule exceptions through the Azure portal, PowerShell, or the Azure CLI v2.

To learn more about working with storage analytics, see [Use Azure Storage analytics to collect logs and metrics data](./storage-analytics.md).

#### [Portal](#tab/azure-portal)

1. Go to the storage account that you want to secure.

2. Select **Networking**.

3. Check that you've chosen to allow access from **Selected networks**.

4. Under **Exceptions**, select the exceptions that you want to grant.

5. Select **Save** to apply your changes.

#### [PowerShell](#tab/azure-powershell)

1. Install [Azure PowerShell](/powershell/azure/install-azure-powershell) and [sign in](/powershell/azure/authenticate-azureps).

2. Display the exceptions for the storage account's network rules:

    ```powershell
    (Get-AzStorageAccountNetworkRuleSet -ResourceGroupName "myresourcegroup" -Name "mystorageaccount").Bypass
    ```

3. Configure the exceptions to the storage account's network rules:

    ```powershell
    Update-AzStorageAccountNetworkRuleSet -ResourceGroupName "myresourcegroup" -Name "mystorageaccount" -Bypass AzureServices,Metrics,Logging
    ```

4. Remove the exceptions to the storage account's network rules:

    ```powershell
    Update-AzStorageAccountNetworkRuleSet -ResourceGroupName "myresourcegroup" -Name "mystorageaccount" -Bypass None
    ```

#### [Azure CLI](#tab/azure-cli)

1. Install the [Azure CLI](/cli/azure/install-azure-cli) and [sign in](/cli/azure/authenticate-azure-cli).

2. Display the exceptions for the storage account's network rules:

    ```azurecli
    az storage account show --resource-group "myresourcegroup" --name "mystorageaccount" --query networkRuleSet.bypass
    ```

3. Configure the exceptions to the storage account's network rules:

    ```azurecli
    az storage account update --resource-group "myresourcegroup" --name "mystorageaccount" --bypass Logging Metrics AzureServices
    ```

4. Remove the exceptions to the storage account's network rules:

    ```azurecli
    az storage account update --resource-group "myresourcegroup" --name "mystorageaccount" --bypass None
    ```

---

## Next steps

Learn more about [Azure network service endpoints](../../virtual-network/virtual-network-service-endpoints-overview.md).

Dig deeper into [Azure Storage security](../blobs/security-recommendations.md).
