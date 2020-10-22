---
title: Manage subscriptions
linkTitle: Manage subscriptions
weight: 10
date: 2019-12-16T00:00:00.000Z
description: This section describes how to manage your AMPLIFY Unified Catalog subscriptions
---
### Objective

Learn how subscriptions work in AMPLIFY Unified Catalog and configure a custom subscription workflow.

### **How subscriptions work**

Subscriptions can be used to secure access to an asset in the AMPLIFY Unified Catalog. Consumers who need to get access to an asset of any type, must subscribe to it first. Under certain conditions, the asset providers might want to disable subscriptions for their asset. As a result, consumers can use the asset without having to subscribe. For instance, when API Providers might want to publish an API that is not secured with an API key.

Asset providers can also configure the subscription metadata that is required from consumers at the time of subscription. This can be done either when publishing the asset to the AMPLIFY Unified Catalog or after the asset has been published.

{{< alert title="Note" color="primary" >}}Enabling or disabling a subscriptions is currently available only through [AMPLIFY Central CLI](https://axway-open-docs.netlify.app/docs/central/cli_central/) or AMPLIFY API.{{< /alert >}}

#### Subscription scope

A subscription is owned by a team. That means, all members of that team can see the subscription details and share the credentials. Subscriptions cannot be shared between different teams and are not visible outside of that team.

The asset providers can only see subscription requests to their catalog assets.

#### Subscription lifecycle

A successful subscription flow looks like this:

1. Subscription is created when a consumer subscribes to an asset.
2. Subscription state is set to **requested**.
3. Subscription request is approved either automatically or manually by the asset provider.
4. Subscription state is set to **approved**.
5. Provision access to the asset for a consumer. For API assets, this can mean generating an API Key.
6. Subscription state is set to **active** once the provisioning is complete. At this point, the consumer can successfully use the asset.

The list below outlines all possible subscription states and their meaning

* **requested** - A transient state that indicates the user has submitted a subscription request and approval is required.
* **approved** - Indicates that the subscription request was approved by the asset provider and the access to the asset is being provisioned.
* **rejected** - Indicates that the subscription request was rejected by the asset provider.
* **active** - Indicates the provisioning is complete and consumer can use the asset. i.e make an API call for API assets or exchange files with partner for MFT assets.
* **failed_to_subscribe** - This state indicates there was an error during the provisioning.
* **unsubscribe_initiated** - Indicates that the consumer has requested to unsubscribe from an asset. Asset providers can also unsubscribe consumers of their assets. At this moment, the access to the asset is being removed.
* **unsubscribed** - Deprovisioning is complete.
* **failed_to_unsubscribe** - This state indicates the request to unsubscribe could not be fulfilled due to an internal error.
* **change_requested** - Indicates that a chance was submitted for a subscription. Change requests can be approved or rejected by asset providers.



The transition between different subscription states can be seen in the diagram below.

![subscription state diagram](/Images/catalog/api-subscription-state-diagram.png "Subscription state diagram")



### View Subscriptions

To view the historical transition of a subscription request:

1. Select **Subscriptions** in the left navigation bar in AMPLIFY Central.
2. Use the top filter drop-down to filter the subscription by their status.
3. Select a subscription. This will bring up the subscription details screen. 
4. Click on the **History** tab. Here you can see the historical activity on a subscription request.

Watch a quick animation to view your subscription history.

![view subscriptions](/Images/catalog/view_subscriptions.gif "view subscriptions")

### Enable manual subscriptions

As an asset provider, you can configure how subscription requests will be approved. There are two modes supported:

* **automatic:** there is no manual intervention required from the asset provider and the user is automatically granted access to consume the asset.
* **manual:** the asset provider manually approves or rejects each requests either from the AMPLIFY Central or by delegating the approval mechanism through an external system. (Eg. Microsoft Teams, Flow Manager)

Follow these steps to enable manual approvals:

1. Select **Catalog** in the left navigation bar. This will bring up the **Explore Catalog** screen where you can browse and discover the asset available for your team.
2. Select your asset.
3. From the **Ellipsis** button, select *Edit*.
4. Select **Manual** in the Subscription approval drop-down.
5. Click **Update** to save your changes.

Below is a screen shot.

![manual subscription](/Images/catalog/manual_subscription.png "Manual subscription")



{{< alert title="Note" color="primary" >}}Only users that are assigned the Central Admin role can change the subscription approval mode.{{< /alert >}}

### Approve / Reject subscriptions

As an asset provider, you can approve or reject subscription requests to assets in the AMPLIFY Unified Catalog from the **Subscription** management screen.

Follow the steps below to approve a subscription:

1. Select **Subscriptions** in the left navigation bar in AMPLIFY Central.
2. Use the top filter drop-down to filter the subscription by their status.
3. Select a subscription that has the status set to **Awaiting Approval.**
4. Review the asset subscription request details.
5. Click **Approve** or **Reject.**
6. You can can choose to provide an optional approval message for the subscriber.

{{< alert title="Note" color="primary" >}}Only users that are assigned the Central Admin role can approve or reject subscriptions.{{< /alert >}}

### Unsubscribe from an asset

To unsubscribe from an asset:

* Select **Subscriptions** from the left navigation bar in AMPLIFY Central.
* Click **Unsubscribe** on the subscription of your choice. You can enter an optional justification for unsubscribing from the asset.

You can also unsubscribe from the Subscription detail page as shown below.

![unsubscribe](/Images/catalog/unsubscribe_asset.gif "Unsubscribe from an asset")

### Delete the subscription of an asset

Deleting a subscription will remove it from the system. You can only delete subscriptions which are in `Unsubscribed` or `Deleted` status.

Watch this quick animation to delete your subscription.

![delete subscription](/Images/catalog/delete_subscription.gif "Delete subscription")