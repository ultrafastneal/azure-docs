---
title: Account resource management with Batch Management .NET | Microsoft Docs
description: Create, delete, and modify Azure Batch account resources with the Batch Management .NET library.
services: batch
documentationcenter: .net
author: tamram
manager: timlt
editor: ''
tags: azure-resource-manager

ms.assetid: 16279b23-60ff-4b16-b308-5de000e4c028
ms.service: batch
ms.devlang: multiple
ms.topic: article
ms.tgt_pltfrm: vm-windows
ms.workload: big-compute
ms.date: 01/20/2017
ms.author: tamram

---
# Manage Azure Batch accounts and quotas with Batch Management .NET
> [!div class="op_single_selector"]
> * [Azure portal](batch-account-create-portal.md)
> * [Batch Management .NET](batch-management-dotnet.md)
> 
> 

You can lower maintenance overhead in your Azure Batch applications by using the [Batch Management .NET][api_mgmt_net] library to automate Batch account creation, deletion, key management, and quota discovery.

* **Create and delete Batch accounts** within any region. If, as an independent software vendor (ISV) for example, you provide a service for your clients in which each is assigned a separate Batch account for billing purposes, you can add account creation and deletion capabilities to your customer portal.
* **Retrieve and regenerate account keys** programmatically for any of your Batch accounts. This can help you comply with security policies that enforce periodic rollover or expiry of account keys. When you have several Batch accounts in various Azure regions, automation of this rollover process increases your solution's efficiency.
* **Check account quotas** and take the trial-and-error guesswork out of determining which Batch accounts have what limits. By checking your account quotas before starting jobs, creating pools, or adding compute nodes, you can proactively adjust where or when these compute resources are created. You can determine which accounts require quota increases before allocating additional resources in those accounts.
* **Combine features of other Azure services** for a full-featured management experience--by using Batch Management .NET, [Azure Active Directory][aad_about], and the [Azure Resource Manager][resman_overview] together in the same application. By using these features and their APIs, you can provide a frictionless authentication experience, the ability to create and delete resource groups, and the capabilities that are described above for an end-to-end management solution.

> [!NOTE]
> While this article focuses on the programmatic management of your Batch accounts, keys, and quotas, you can perform many of these activities by using the [Azure portal][azure_portal]. For more information, see [Create an Azure Batch account using the Azure portal](batch-account-create-portal.md) and [Quotas and limits for the Azure Batch service](batch-quota-limit.md).
> 
> 

## Create and delete Batch accounts
As mentioned, one of the primary features of the Batch Management API is to create and delete Batch accounts in an Azure region. To do so, use [BatchManagementClient.Account.CreateAsync][net_create] and [DeleteAsync][net_delete], or their synchronous counterparts.

The following code snippet creates an account, obtains the newly created account from the Batch service, and then deletes it. In this snippet and the others in this article, `batchManagementClient` is a fully initialized instance of [BatchManagementClient][net_mgmt_client].

```csharp
// Create a new Batch account
await batchManagementClient.Account.CreateAsync("MyResourceGroup",
    "mynewaccount",
    new BatchAccountCreateParameters() { Location = "West US" });

// Get the new account from the Batch service
AccountResource account = await batchManagementClient.Account.GetAsync(
    "MyResourceGroup",
    "mynewaccount");

// Delete the account
await batchManagementClient.Account.DeleteAsync("MyResourceGroup", account.Name);
```

> [!NOTE]
> Applications that use the Batch Management .NET library and its BatchManagementClient class require **service administrator** or **coadministrator** access to the subscription that owns the Batch account to be managed. For more information, see the [Azure Active Directory](#azure-active-directory) section and the [AccountManagement][acct_mgmt_sample] code sample.
> 
> 

## Retrieve and regenerate account keys
Obtain primary and secondary account keys from any Batch account within your subscription by using [ListKeysAsync][net_list_keys]. You can regenerate those keys by using [RegenerateKeyAsync][net_regenerate_keys].

```csharp
// Get and print the primary and secondary keys
BatchAccountListKeyResult accountKeys =
    await batchManagementClient.Account.ListKeysAsync(
        "MyResourceGroup",
        "mybatchaccount");
Console.WriteLine("Primary key:   {0}", accountKeys.Primary);
Console.WriteLine("Secondary key: {0}", accountKeys.Secondary);

// Regenerate the primary key
BatchAccountRegenerateKeyResponse newKeys =
    await batchManagementClient.Account.RegenerateKeyAsync(
        "MyResourceGroup",
        "mybatchaccount",
        new BatchAccountRegenerateKeyParameters() {
            KeyName = AccountKeyType.Primary
            });
```

> [!TIP]
> You can create a streamlined connection workflow for your management applications. First, obtain an account key for the Batch account you wish to manage with [ListKeysAsync][net_list_keys]. Then, use this key when initializing the Batch .NET library's [BatchSharedKeyCredentials][net_sharedkeycred] class, which is used when initializing [BatchClient][net_batch_client].
> 
> 

## Check Azure subscription and Batch account quotas
Azure subscriptions and the individual Azure services like Batch all have default quotas that limit the number of certain entities within them. For the default quotas for Azure subscriptions, see [Azure subscription and service limits, quotas, and constraints](../azure-subscription-service-limits.md). For the default quotas of the Batch service, see [Quotas and limits for the Azure Batch service](batch-quota-limit.md). By using the Batch Management .NET library, you can check these quotas in your applications. This enables you to make allocation decisions before you add accounts or compute resources like pools and compute nodes.

### Check an Azure subscription for Batch account quotas
Before creating a Batch account in a region, you can check your Azure subscription to see whether you are able to add an account in that region.

In the code snippet below, we first use [BatchManagementClient.Account.ListAsync][net_mgmt_listaccounts] to get a collection of all Batch accounts that are within a subscription. Once we've obtained this collection, we determine how many accounts are in the target region. Then we use [BatchManagementClient.Subscriptions][net_mgmt_subscriptions] to obtain the Batch account quota and determine how many accounts (if any) can be created in that region.

```csharp
// Get a collection of all Batch accounts within the subscription
BatchAccountListResponse listResponse =
        await batchManagementClient.Account.ListAsync(new AccountListParameters());
IList<AccountResource> accounts = listResponse.Accounts;
Console.WriteLine("Total number of Batch accounts under subscription id {0}:  {1}",
    creds.SubscriptionId,
    accounts.Count);

// Get a count of all accounts within the target region
string region = "westus";
int accountsInRegion = accounts.Count(o => o.Location == region);

// Get the account quota for the specified region
SubscriptionQuotasGetResponse quotaResponse = await batchManagementClient.Subscriptions.GetSubscriptionQuotasAsync(region);
Console.WriteLine("Account quota for {0} region: {1}", region, quotaResponse.AccountQuota);

// Determine how many accounts can be created in the target region
Console.WriteLine("Accounts in {0}: {1}", region, accountsInRegion);
Console.WriteLine("You can create {0} accounts in the {1} region.", quotaResponse.AccountQuota - accountsInRegion, region);
```

In the snippet above, `creds` is an instance of [TokenCloudCredentials][azure_tokencreds]. To see an example of creating this object, see the [AccountManagement][acct_mgmt_sample] code sample on GitHub.

### Check a Batch account for compute resource quotas
Before increasing compute resources in your Batch solution, you can check to ensure the resources you want to allocate won't exceed the account's quotas. In the code snippet below, we print the quota information for the Batch account named `mybatchaccount`. In your own application, you could use such information to determine whether the account can handle the additional resources to be created.

```csharp
// First obtain the Batch account
BatchAccountGetResponse getResponse =
    await batchManagementClient.Account.GetAsync("MyResourceGroup", "mybatchaccount");
AccountResource account = getResponse.Resource;

// Now print the compute resource quotas for the account
Console.WriteLine("Core quota: {0}", account.Properties.CoreQuota);
Console.WriteLine("Pool quota: {0}", account.Properties.PoolQuota);
Console.WriteLine("Active job and job schedule quota: {0}", account.Properties.ActiveJobAndJobScheduleQuota);
```

> [!IMPORTANT]
> While there are default quotas for Azure subscriptions and services, many of these limits can be raised by issuing a request in the [Azure portal][azure_portal]. For example, see [Quotas and limits for the Azure Batch service](batch-quota-limit.md) for instructions on increasing your Batch account quotas.
> 
> 

## Batch Management .NET, Azure AD, and Resource Manager
When you work with the Batch Management .NET library, you typically also use [Azure Active Directory][aad_about] (Azure AD) and the [Azure Resource Manager][resman_overview]. The sample project discussed below uses both Azure Active Directory and the Resource Manager while it demonstrates the Batch Management .NET API.

### Azure Active Directory
Azure itself uses Azure AD for the authentication of its customers, service administrators, and organizational users. In the context of Batch Management .NET, you use Azure AD to authenticate a subscription administrator or co-administrator. This allows the management library to query the Batch service and perform the operations described in this article.

In the sample project discussed below, the Azure [Active Directory Authentication Library][aad_adal] (ADAL) is used to prompt the user for their Microsoft credentials. When service administrator or coadministrator credentials are supplied, the application can query Azure for a list of subscriptions--and can create and delete both resource groups and Batch accounts.

### Resource Manager
When you create Batch accounts with the Batch Management .NET library, you will typically be creating them within a [resource group][resman_overview]. You can create the resource group programmatically by using the [ResourceManagementClient][resman_client] class in the [Resource Manager .NET][resman_api] library. Or you can add an account to an existing resource group that you created previously by using the [Azure portal][azure_portal].

## Sample project on GitHub
To see Batch Management .NET in action, check out the [AccountManagment][acct_mgmt_sample] sample project on GitHub. This console application shows the creation and usage of [BatchManagementClient][net_mgmt_client] and [ResourceManagementClient][resman_client]. It also demonstrates the usage of the Azure [Active Directory Authentication Library][aad_adal] (ADAL), which is required by both clients.

To run the sample application successfully, you must first register it with Azure AD by using the Azure portal. Follow the steps in the [Adding an Application](../active-directory/develop/active-directory-integrating-applications.md#adding-an-application) section in [Integrating applications with Azure Active Directory][aad_integrate] to register the sample application within your own account's Default Directory. Be sure to select **Native Client Application** for the type of application, and you can specify any valid URI (such as `http://myaccountmanagementsample`) for the **Redirect URI**--it does not need to be a real endpoint.

After adding your application, delegate the **Access Azure Service Management as organization** permission to the *Windows Azure Service Management API* application in the application's settings in the portal:

![Application permissions in Azure portal][2]

> [!TIP]
> If **Windows Azure Service Management API** does not appear under *permissions to other applications*, click **Add application**, select **Windows Azure Service Management API**, then click the check mark button. Then, delegate permissions as specified above.
> 
> 

Once you've added the application as described above, update `Program.cs` in the [AccountManagment][acct_mgmt_sample] sample project with your application's Redirect URI and Client ID. You can find these values in the **Configure** tab of your application:

![Application configuration in Azure portal][3]

The [AccountManagment][acct_mgmt_sample] sample application demonstrates the following operations:

1. Acquire a security token from Azure AD by using [ADAL][aad_adal]. If the user is not already signed in, they are prompted for their Azure credentials.
2. By using the security token obtained from Azure AD, create a [SubscriptionClient][resman_subclient] to query Azure for a list of subscriptions associated with the account. This allows selection of a subscription if multiple are found.
3. Create a credentials object associated with the selected subscription.
4. Create [ResourceManagementClient][resman_client] by using the credentials.
5. Use [ResourceManagementClient][resman_client] to create a resource group.
6. Use [BatchManagementClient][net_mgmt_client] to perform several Batch account operations:
   * Create a Batch account in the new resource group.
   * Get the newly created account from the Batch service.
   * Print the account keys for the new account.
   * Regenerate a new primary key for the account.
   * Print the quota information for the account.
   * Print the quota information for the subscription.
   * Print all accounts within the subscription.
   * Delete newly created account.
7. Delete the resource group.

Before deleting the newly created Batch account and resource group, you can inspect both in the [Azure portal][azure_portal]:

![Azure portal displaying the resource group and Batch account][1]
<br />
*Azure portal displaying new resource group and Batch account*

[aad_about]: ../active-directory/active-directory-whatis.md "What is Azure Active Directory?"
[aad_adal]: ../active-directory/active-directory-authentication-libraries.md
[aad_auth_scenarios]: ../active-directory/active-directory-authentication-scenarios.md "Authentication Scenarios for Azure AD"
[aad_integrate]: ../active-directory/active-directory-integrating-applications.md "Integrating Applications with Azure Active Directory"
[acct_mgmt_sample]: https://github.com/Azure/azure-batch-samples/tree/master/CSharp/AccountManagement
[api_net]: http://msdn.microsoft.com/library/azure/mt348682.aspx
[api_mgmt_net]: https://msdn.microsoft.com/library/azure/mt463120.aspx
[azure_portal]: http://portal.azure.com
[azure_storage]: https://azure.microsoft.com/services/storage/
[azure_tokencreds]: https://msdn.microsoft.com/library/azure/microsoft.windowsazure.tokencloudcredentials.aspx
[batch_explorer_project]: https://github.com/Azure/azure-batch-samples/tree/master/CSharp/BatchExplorer
[net_batch_client]: https://msdn.microsoft.com/library/azure/microsoft.azure.batch.batchclient.aspx
[net_list_keys]: https://msdn.microsoft.com/library/azure/microsoft.azure.management.batch.accountoperationsextensions.listkeysasync.aspx
[net_create]: https://msdn.microsoft.com/library/azure/microsoft.azure.management.batch.accountoperationsextensions.createasync.aspx
[net_delete]: https://msdn.microsoft.com/library/azure/microsoft.azure.management.batch.accountoperationsextensions.deleteasync.aspx
[net_regenerate_keys]: https://msdn.microsoft.com/library/azure/microsoft.azure.management.batch.accountoperationsextensions.regeneratekeyasync.aspx
[net_sharedkeycred]: https://msdn.microsoft.com/library/azure/microsoft.azure.batch.auth.batchsharedkeycredentials.aspx
[net_mgmt_client]: https://msdn.microsoft.com/library/azure/microsoft.azure.management.batch.batchmanagementclient.aspx
[net_mgmt_subscriptions]: https://msdn.microsoft.com/library/azure/microsoft.azure.management.batch.batchmanagementclient.subscriptions.aspx
[net_mgmt_listaccounts]: https://msdn.microsoft.com/library/azure/microsoft.azure.management.batch.iaccountoperations.listasync.aspx
[resman_api]: https://msdn.microsoft.com/library/azure/mt418626.aspx
[resman_client]: https://msdn.microsoft.com/library/azure/microsoft.azure.management.resources.resourcemanagementclient.aspxs
[resman_subclient]: https://msdn.microsoft.com/library/azure/microsoft.azure.subscriptions.subscriptionclient.aspx
[resman_overview]: ../azure-resource-manager/resource-group-overview.md

[1]: ./media/batch-management-dotnet/portal-01.png
[2]: ./media/batch-management-dotnet/portal-02.png
[3]: ./media/batch-management-dotnet/portal-03.png
