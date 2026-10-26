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

## Examples

Taken from this package's own tests, which compile and run on every release.

```osy
using Osysharp.Shipping;

[Test]
void a_flat_rate_is_free_above_its_threshold_and_only_where_it_delivers() {
  var flat = new FlatRateDelivery { Price = 900, FreeFrom = 20000, Countries = ["SE", "NO"] };
  Assert.Equal(900, flat.Quote(Basket(5000, "SE"))[0].Price);
  Assert.Equal(0, flat.Quote(Basket(25000, "se"))[0].Price);
  Assert.Empty(flat.Quote(Basket(5000, "DE")));
  Assert.Single(flat.Quote(Basket(5000, null)));   // before an address, it is still on offer
}
```

From [`tests/delivery.test.osy`](tests/delivery.test.osy).

```osy
using Osysharp.Shipping;

[Test]
void every_ready_provider_offers_its_options_and_collection_needs_no_address() {
  List<IDeliveryProvider> providers = [
    new FlatRateDelivery { Price = 900 },
    new CollectInStore { Address = "Storgatan 1, weekdays 10–18" },
  ];
  var options = Options(providers, Basket(5000, "SE"));
  Assert.Equal(2, options.Count);
  Assert.True(options[0].NeedsAddress);
  Assert.False(options[1].NeedsAddress);
  Assert.Equal("Storgatan 1, weekdays 10–18", options[1].Description);
}
```

From [`tests/delivery.test.osy`](tests/delivery.test.osy).

```osy
using Osysharp.Shipping;

[Test]
void zones_price_by_country_and_weight_and_the_rest_of_the_world_catches_the_others() {
  var zones = new ZoneRateDelivery { Zones = [
    new DeliveryZone { Name = "Sweden", Countries = ["SE"], FreeFrom = 100000,
                       Rates = [ new WeightRate { UpToGrams = 2000, Price = 4900 }, new WeightRate { UpToGrams = 10000, Price = 9900 } ] },
    new DeliveryZone { Name = "World", Rates = [ new WeightRate { UpToGrams = 5000, Price = 24900 } ] },
  ] };
  var light = Basket(5000, "SE"); light.WeightGrams = 1500;
  Assert.Equal(4900, zones.Quote(light)[0].Price);
  var heavy = Basket(5000, "SE"); heavy.WeightGrams = 6000;
  Assert.Equal(9900, zones.Quote(heavy)[0].Price);
  var dear = Basket(150000, "SE"); dear.WeightGrams = 6000;
  Assert.Equal(0, zones.Quote(dear)[0].Price);                 // free in Sweden over 1 000
  var far = Basket(5000, "JP"); far.WeightGrams = 1000;
  Assert.Equal("World", zones.Quote(far)[0].Description);
  var tooHeavy = Basket(5000, "JP"); tooHeavy.WeightGrams = 9000;
  Assert.Empty(zones.Quote(tooHeavy));                         // over the World ladder: not delivered there
  Assert.Empty(zones.Quote(Basket(5000, null)));               // no address yet: no price to give
}
```

From [`tests/delivery.test.osy`](tests/delivery.test.osy).

## Learn more

- [Osysharp.Shipping in the package catalogue](https://osysharp.com/packages/#osysharp-shipping)
- [The Osy# reference](https://osysharp.com/reference/) — the language, the runtime and every kit
- [`llms-full.txt`](https://osysharp.com/llms-full.txt) — the whole reference and specification as one Markdown file, for a coding agent

## Licence

MIT — see [`LICENSE`](LICENSE).
