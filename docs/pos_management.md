# <a name="pos_management"></a>Point-of-Sale Management
The Point-of-Sale (PoS) represents the contact point between a customer wanting to pay with MobilePay and the merchant.
To initiate a MobilePay payment it is necessary to first create a PoS. 

## Onboarding
Each PoS belongs to a *Store* which in turn is associated with a *Brand*. A brand can be thought of as a combination of a name and a logo. When a MobilePay customer checks in on a PoS they will see the brand name and the logo in the app, which helps the customer confirm that they have in fact checked in where they intended. An example of a brand could be 7-Eleven in Denmark or K-Market in Finland. A brand is identified by a ``merchantBrandId``. Each brand consists of one or more stores. Each store also has a name which is also shown to the customer when they check in on a PoS that belongs to that store. A ``merchantLocationId`` together with a ``merchantBrandId`` identifies a store within a brand. 

Brands and stores are created by the merchant when onboarding with MobilePay PoS and the merchant will typically provide the ``merchantBrandId``s and ``merchantLocationId``s for the merchant's brands and stores to the integrator.

When the integrator has received the ``merchantBrandId`` and the ``merchantLocationId`` they will have to call ``GET /v10/stores`` with the two ids, and in return they will receive a ``storeId`` which will be used to create all the PoSes on that store. The ``storeId`` will therefore have to be persisted in an application configuration file for subsequent calls to the V10 API. Below diagram illustrates a flow for getting the ``storeId`` using ``GET /v10/stores``.

[![](assets/images/get_store.png)](assets/images/get_store.png)

### Legacy lookup endpoint
To help integrators with migration of PoSes, we introduced an endpoint allowing for a lookup of merchant vat number by `legacyMerchantId` and `locationId`. It's is available under `GET https://api.mobilepay.dk/pos-core-for-integrators/public/v1/stores/vat`, having two query parameters: `legacyMerchantId` and `locationId`

This endpoint is temporary, meant only to help with migration of PoSes to V10. It will be removed in 2021Q2.

## <a name="pos_creation"></a> PoS Creation
A PoS is created using the ````POST /v10/pointofsales```` endpoint. A PoS is identified in the PoS V10 API by a ````posId```` that is assigned by MobilePay upon creation of the PoS. Clients can provide their own internal identifier as a ````merchantPosId```` upon creation and use the ````GET /v10/pointofsales```` endpoint to lookup a ````posId```` based on a ````merchantPosId````. The `merchantPosId` field is required, so if no internal identifier is applicable, the client should generate and supply a random string (eg. a fresh GUID) instead.

### <a name="beacons"></a> Beacons
The first thing to consider when creating a PoS is what beacon(s) will be used to connect MobilePay users to the given PoS.
This can range from an unmanned vending machine that has no payment hardware at all and hence only shows a QR code on a screen, to a full fledged super market ECR with a 2-way bluetooth capable terminal that also can show a QR code. To create a PoS, the client needs to provide a list of possible ways to detect the PoS. The more accurate the list is, the better MobilePay will be able to detect errors (if bluetooth is provided as a beacon type but we detect that no user ever checks in via bluetooth something is likely wrong). It is recommended to keep the list of supported beacon types in an application configuration and then edit the list in case the setup changes.

Each beacon, whether through a MobilePay QR code or a bluetooth/NFC signal, encodes a ````beaconId```` that can be read by the MobilePay app. It is the ````beaconId```` that is used to connect a MobilePay user to a specific PoS. ````beaconId````s are globally unique across all merchants in MobilePay PoS and each ````beaconId```` can refer to at most one active PoS at any given time. 

Depending on the client setup, there are different use cases for handling ````beaconId````s in API V10

#### Client that only supports dynamic QR codes
In case the client only allows QR beacons (no physical device) and can create a QR code dynamically (i.e generate a QR code and show it on a screen in opposition to printing a physical QR code), then the client can choose to let MobilePay create a GUID to use as ````beaconId````. The client then omits to provide a ````beaconId```` on PoS creation and afterwards queries the PoS to get the ````beaconId````. The client can then store the ````beaconId```` in memory for QR code generation. Everytime the client reboots the client then has to query the PoS and grab the ````beaconId````. This way the client is not required to store a ````beaconId```` in a configuration file since they can rely on querying it dynamically.

#### Client that only supports static QR codes
In case the client only allows QR beacons but is not able to generate a QR code dynamically, the client should generate the ````beaconId```` and provide it on PoS creation. The ````beaconId```` should then be stored locally in a configuration file so that it can be used if the PoS needs to be updated (i.e. deleted and re-created. See [PoS Update and Deletion](pos_management#pos_updating_deletion)). To avoid clashes, the client must use a GUID as the ````beaconId````.

#### Client that supports physical devices (terminals, MobilePay white boxes)
In cases where the client uses a physical device then that device will have a MobilePay ````beaconId```` associated with it. On PoS creation this ````beaconId```` has to be provided. Some devices allows a client to read the ````beaconId```` from it. If that is the case we recommend to read the ````beaconId```` when the client reboots and query the PoS to see if the ````beaconId````s match. If not delete the PoS and re-create it with the new ````beaconId````. This will make it possible to replace the device if its broken, and only have to reboot the system to propagate the changes.
If reading the ````beaconId```` from the device is not possible, we recommend to store the ````beaconId```` locally in a configuration file so that it persists through reboots.

### <a name="callback"></a>Callback
If the client system cannot detect when a MobilePay user wants to pay and therefore needs to use the [Notification service](detecting_mobilePay#notification), the client should set the callback parameter accordingly when calling ````POST /v10/pointofsales````.
It is recommended to store the callback alias in the config file of the application.

### Naming
The last thing to keep in mind when creating a PoS is to consider the name. When a MobilePay user checks in on the PoS they will in the app see, in sequence: The name of the brand, the name of the store and the name of the PoS. We recommend naming the PoS so that the MobilePay user can verify that they in fact have checked in the right place. So in a supermarket scenario a good name for the PoS would be "Check-out 1" for the first check-out counter in that supermarket.

## <a name="pos_updating_deletion"></a>PoS Updating and Deletion

A PoS can be deleted using the ````DELETE /v10/pointofsales/{posId}```` endpoint.

We recommend only deleting a PoS if it is either not going to be used anymore, or you need to update it to reflect changes like a new callback alias, new name, new ````beaconId```` etc.

When a PoS is deleted it is no longer possible to issue payments. However it will still be possible to capture or cancel payments that are in the reserved state. It is best practice to delay the deletion of a PoS until all payments have either been cancelled or captured.

## Keeping in sync with MobilePay

### When PoS reboots
When the client reboots it is good practice to query the PoS with ````GET /v10/pointofsales```` with the ````merchantPosId```` and persist the ````posId```` in memory to use for initiating payments. If no PoS is returned, the client will have to re-create it. Here is the flow described:
[![](assets/images/PoS_Onboarding.png)](assets/images/PoS_Onboarding.png)


We recommend the client to store the following in a configuration file to be able to create the PoS when needed:

* ````StoreId````
* ````MerchantPosId````
* Name of PoS
* ````BeaconId```` (unless it can be read from the device itself. See [Beacons](pos_management#beacons) )
* Callback (If the client is dependent on the notification service. See [Callback](pos_management#callback))
* Supported beacon types

## <a name="master-data"></a>Master Data

A PoS belongs to a store which in turn belongs to a merchant and associated with a brand. A PoS is identified by a ````posId````, but it is also possible to refer to a PoS by its ````beaconId```` or ````merchantPosId````. There can be at most one active PoS with a given ````beaconId```` at any given time. There can be at most one active PoS with a given ````merchantPosId```` at any given time, for a given integrator and merchant. 

A store is identified by a ````storeId````, but it is also possible to refer to a store by the combination of ````merchantBrandId```` and ````merchantLocationId````. Two stores with the same ````merchantLocationId```` but different ````merchantBrandId````s are not related in any way. The ````merchantBrandId````, ````merchantLocationId```` and ````storeId```` are supplied by MobilePay when the Merchant/Store is onboarded. 
