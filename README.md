# Gift Registry Import - How It Works

Runbook for importing Gift Registry data into Swym for a Shopify store, based on the working test import in this folder.

## 1. Requirements

- Node.js v20.6+ (for `node --env-file=.env`). This project was run on v24.
- `@fast-csv/parse` (installed via `npm install` in this folder).
- Credentials for the **target** store:
  - `SHOP` - the target store's `.myshopify.com` domain.
  - `ADMIN_API_ACCESS_TOKEN` - Shopify Admin API access token (`shpat_...`) for the target store.
  - `APP_ACCESS_TOKEN` - Swym's AppAccessToken for the merchant.
  - `Pid` - Swym's merchant/store identifier, used per registry row.

## 2. Where the keys come from - Metabase "Merchant configs" table

For any given merchant/store, its Pid and AppAccessToken (and other Swym config) can be looked up directly in the **Merchant configs** table in Metabase. Use this as the source of truth when:
- setting up `.env` for a store you don't already have credentials for, or
- confirming the `Pid` in an exported CSV actually matches the target merchant, since a CSV's `Pid` column can be stale or come from a different environment than the one you're importing into.

## 3. Files needed

- **Combined registry CSV** (one row per registry), e.g. `All_registries_aura.csv`. Must include, at minimum: `RegistryId`, `RegistryName`, `CreatorName`, `CoCreatorName`, `Email`, `Description`, `ExpiryDate`, `Occasion`, `Mode`, `Password`, `Settings -> IsPasswordProtected`, `IsDeleted`, `IsArchived`, `Pid`, and the `Address -> *` columns (FirstName, LastName, Address1/2, City, Province, Country, Zip, Phone, Company, Street).
- **Per-registry item CSV**, named `<RegistryId>.csv` (one file per registry, one row per item), with columns: `product id(empi)`, `variant id (epi)`, `Handle`, `RegistryId`, `AskQuantity`, `BoughtQuantity`.
- Both must sit where `COMBINED_REGISTRY_CSV` / `INDIVIDUAL_REGISTRY_DIR` in `.env` point to.

Column names are not universal - they were re-derived from the actual export files for this dataset, and differed from the reference script's original sample data (e.g. `RegistryId` vs `ID`). Always confirm real headers before reusing this mapping for a new export.

## 4. The script

`src/importAuraRegistry.js`, per registry row:
1. Looks up (or creates) a Shopify customer by email on the **target** store.
2. Calls Swym's `generateAuth` with that store's `Pid` + `APP_ACCESS_TOKEN` to get a shopper JWT.
3. Looks for an existing registry matching name + email + expiry date; reuses it if found, otherwise creates one.
4. Reads that registry's item CSV and, for each item, adds the product (`epi`/`empi`/`du`), sets its `askQuantity`, and marks any already-bought quantity.

## 5. Critical: map epi/empi/du to the target store, not the source

`product id(empi)`, `variant id (epi)`, and `Handle` in the item CSV are the **source** store's Shopify product/variant IDs and product URL. These are only valid to import as-is if the source and target store are the **same** store - which is what made this test import (`swymtest-aura-divya.myshopify.com` to itself) work directly.

For a real migration into a **different** target store:
- The source `empi`/`epi` values will not exist in the target store's catalog and must not be used as-is.
- Before import, join each item row to the target store's product catalog (matching on SKU, handle, or title - not on ID) to find the corresponding target `empi`/`epi`.
- Rebuild `du` from the target store's own domain and the target product's handle: `https://<target-shop>/products/<target-handle>`, not the source URL from the export.

Skipping this step will attach registries to products that don't exist (or worse, to the wrong products) on the target store.

## 6. Running it

```
cp .env.example .env      # fill SHOP / ADMIN_API_ACCESS_TOKEN / APP_ACCESS_TOKEN
npm install
npm run import:aura:dry-run   # checks only, no writes
npm run import:aura            # live import, writes import_results.json
```

## 7. Verifying it actually worked

Don't rely on the script's own log alone:
1. `import_results.json` - the script's reported success/failed/skipped counts.
2. `npm run verify:aura` - a live probe (`src/verifyImport.js`) that re-authenticates as the shopper and fetches the registry back directly from Swym's API, independent of the import script.
3. **Metabase** - check the registries data (and the Merchant configs table, to confirm the Pid/keys used were correct for that merchant) to confirm the row is present in the merchant's real data, not just returned by the API.

All three agreed for this test import: registry `2050360` -> Swym registry `2051090`, confirmed via the API probe and independently confirmed in Metabase.
