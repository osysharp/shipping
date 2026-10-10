# Osysharp.Shipping — one contract for getting goods to a customer, whichever carrier carries them

## Use case

A shop sends parcels, and how it sends them changes: a flat rate to start with, collection from the shop, then a
carrier such as PostNord, or Sendcloud with many carriers behind one integration. This package is the contract
between the app that sends and the carriers that carry. The app depends on it and on no carrier in particular; each
way of delivering is a class implementing `IDeliveryProvider` — it quotes the options for a basket and an address,
books the shipment when the goods go out, and says where the parcel is.

It ships the two ways that need no carrier service: `FlatRateDelivery` (a fixed price, optionally free above an
amount and limited to some countries) and `CollectInStore`.

## Install and use

```osy
app MyShop {
  use Osysharp.Shipping@0;
  …
}
```

With `Osysharp.Shop`, the ways to deliver are the shop's list:

```osy
app.Shop = new ShopSetup {
  …
  Delivery = [
    new CollectInStore   { Label = "Collect from the shop" },
    new FlatRateDelivery { Label = "Home delivery", Price = 4900, FreeFrom = 50000, DaysMin = 2, DaysMax = 4 },
  ],
};
```

## The contract

An `IDeliveryProvider` answers:

| member | what it does |
|---|---|
| `IsReady()` | whether it can be offered now — its credentials are set |
| `Quote(DeliveryQuoteRequest)` | the options for a basket and an address, with prices and days |
| `PickupPoints(optionId, near)` | where a parcel can be collected, for an option that delivers to a point |
| `Ship(ShipmentRequest)` | book the shipment; answers the tracking number and the label |
| `BookReturn`, `BookPickup` | a return label, a collection from the shop (when the carrier offers them) |
| `Track(trackingNumber)` | where the parcel is |
| `HandleEvent(body, headers)` | read and verify a carrier's webhook |

Prices are in minor units (öre, cents).

Published carriers: `Osysharp.Carriers.PostNord`, `Osysharp.Carriers.Sendcloud`, `Osysharp.Carriers.Shippo`.

## Source and package

- Source: `model/Contract.osy` (the contract and its records) and `model/BuiltIn.osy` (`FlatRateDelivery`,
  `CollectInStore`).
- Tests: `tests/`.
- Package: `use Osysharp.Shipping@0;` — `osy lock` fetches the release archive from this repository's releases.

To add a carrier of your own, implement `IDeliveryProvider` in your app (or in a package of your own) and add an
instance to the list.
