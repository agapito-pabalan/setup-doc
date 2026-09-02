# Local Parcel Creation with Staging Shipium

Guide for testing the full end-to-end flow locally: parcel creation (label via Shipium) + rating.

## Prerequisites

1. Run `mix ecto.setup` (fresh DB) or `mix run priv/repo/seeds.exs` (existing DB) to seed the rating engine configuration
2. Have access to the staging environment (for Shipium credentials)

## What the Seeds Provide

After running seeds, you have a complete rating pipeline ready to use:

| Entity | Value |
|--------|-------|
| Shipper | Test Shipper 1 (code: TST01, tenant: simulation-tenant-id) |
| Carrier | test_fedex (Test FedEx) |
| Service Method | fedex-home-delivery-service-method |
| Service Level | Ground |
| Fulfillment Center | TEST-FC-1 (Atlanta, GA 30349) |
| Zone | 90000-90999 → zone code 5 |
| Published Rate | $10.00 flat (zone 5, 0-150 lbs) |
| Sell Rate | $8.50 flat (zone 5, 0-150 lbs) |
| Dimensional Factor | 139 (threshold 1728) |
| Extended Area (DAS) | zip 90210, country US → $5.00 surcharge |
| Carrier Accessorials | DAS, FedEx residential, FedEx fuel — all mapped |
| Accessorial Configs | DAS (use_shipper_rate), Residential & Fuel (use_buy_rate 1.0x) |

## Setup

Create or update `.env.local` with staging Shipium credentials:

```bash
SHIPIUM_API_KEY=<staging_shipium_api_key>
SHIPIUM_URL=https://api.shipium.com
SHIPIUM_TEST_MODE="true"
```

To get the API key, connect to a staging console and run:

```elixir
System.get_env("SHIPIUM_API_KEY")
```

Restart the server after changing `.env.local`.

## Creating a Parcel via UI

In the Orion UI, go to **Parcel > Create Parcel**:

| Field | Value |
|-------|-------|
| Shipper | Test Shipper 1 |
| Fulfillment Center | TEST-FC-1 |
| To Address Type | **Residential** (FedEx Home Delivery only delivers to residential) |
| To Address Zip | 90210 (triggers DAS surcharge) |
| To Address Country | US |
| To Address City/State/Line1 | Any valid values |
| Weight | Any value between 0-150 lbs |
| Dimensions | Any positive values |

### Expected Rating Result

- **Base rate**: $8.50 (Flat)
- **Residential Surcharge**: pass-through from Shipium buy rate
- **Fuel Surcharge**: pass-through from Shipium buy rate
- **DAS**: $5.00 (extended area surcharge for zip 90210, US)

## Rating the Seed Parcel

The seed inserts a sample parcel. To rate it without going through Shipium:

1. Open the parcel in Orion UI
2. Click **Initial Rating**
3. **Refresh the page** — the rating runs asynchronously (Oban worker), so sell rates appear after a page refresh

Expected result: Base $8.50 + DAS $5.00 = **$13.50**

To get the full rating with buy-side accessorials (Residential, Fuel), create a parcel through the UI instead (see above).

## FAQ

### Shipium tenant not found

The tenant is resolved in order: `channel.shipium_label_tenant` → `shipper.shipium_label_tenant` → `shipper.external_reference`. The seed sets `shipium_label_tenant` on Test Shipper 1 by default. If running on old data, update it:

```elixir
ParcelService.Shipment.Shipper
|> Repo.get_by!(name: "Test Shipper 1")
|> ParcelService.Shipment.Shipper.changeset(%{shipium_label_tenant: "simulation-tenant-id"})
|> Repo.update!()
```

To find other available tenants, query staging:

```elixir
import Ecto.Query

Repo.all(
  from s in ParcelService.Shipment.Shipper,
    where: not is_nil(s.shipium_label_tenant),
    select: {s.name, s.id, s.shipium_label_tenant},
    limit: 10
)
```

### All carrier service methods were filtered

The service method name must exist in Shipium's catalog for the tenant. Query staging for valid names:

```elixir
import Ecto.Query
tenant_shipper_id = "<shipper_id_with_the_tenant>"

Repo.all(
  from sm in ParcelService.CarrierServices.ServiceMethod,
    where: sm.shipper_id == ^tenant_shipper_id and is_nil(sm.deleted_at),
    select: sm.name
)
```

Then rename the local service method to match:

```elixir
ParcelService.CarrierServices.ServiceMethod
|> Repo.get_by!(name: "fedex-home-delivery-service-method")
|> ParcelService.CarrierServices.ServiceMethod.changeset(%{name: "<new_name>"}) 
|> Repo.update!()
```

### Service method does not deliver to destination type

FedEx Home Delivery delivers to **residential** only. FedEx Ground delivers to **commercial** only. Make sure the To Address Type in the form matches.

### Fulfillment center not recognized by Shipium

The FC address must match one that Shipium knows about for the tenant. Query staging for valid FCs:

```elixir
import Ecto.Query
tenant_shipper_id = "<shipper_id_with_the_tenant>"

Repo.all(
  from fc in ParcelService.Shipment.FulfillmentCenter,
    join: scf in ParcelService.Shipment.ShipperCarrierFulfillment,
      on: scf.fulfillment_center_id == fc.id,
    where: scf.shipper_id == ^tenant_shipper_id and is_nil(fc.deleted_at),
    select: {fc.alias, fc.address}
)
```

Then update the local FC and make sure the zone group's `zip_start`/`zip_end` covers the new postal code.

### missing_accessorial_mapping_on_carrier_accessorial

A buy-side accessorial name from Shipium has no matching CarrierAccessorial with an `accessorial_id`. The rating engine auto-creates unmatched CarrierAccessorials (without `accessorial_id`). Find them and map them:

```elixir
import Ecto.Query
carrier_id = "<carrier_id>"

Repo.all(
  from ca in ParcelService.RatingEngine.Accessorials.CarrierAccessorial,
    where: ca.carrier_id == ^carrier_id and is_nil(ca.accessorial_id) and is_nil(ca.deleted_at),
    select: {ca.id, ca.name}
)
```

### rate_not_found_missing_discount_rate / rate_not_found_missing_published_rate

The rating engine needs both a published rate card and a shipper sell rate card with matching `service_level_id`, `zone_code`, and weight range. Re-run seeds or verify the records exist.

### accessorial_config_not_found

Every accessorial that gets mapped via CarrierAccessorial needs an AccessorialConfig with a matching weight range. Re-run seeds or create one manually.

### Unable to find a dimensional factor

No DimensionalFactor covers the parcel's weight. Re-run seeds (covers 0-150 lbs) or verify the parcel weight is within range.

### no_shipper_code

The shipper is missing the `code` field. The seed sets `code: "TST01"` by default. If running on old data, update the shipper manually.