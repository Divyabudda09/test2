# Gift Registry Import

> **Swym · Gift Registry · **

---

## 01 · Prerequisites

### Files

- **Combined registry CSV**: one file, one row per registry.
- **Individual registry item CSVs**: one file per registry, named `<RegistryId>.csv`. Use that registry's own `RegistryId` from the combined file as the filename.

### Keys

All four keys are store-specific. Look them up in the Metabase [**Merchant configs**](https://metabase-prod.ops.internalswym.com/question/1127-merchantconfigs) table for the store you are importing into.

| Key | What it is |
|---|---|
| `SHOP` | Target store's `.myshopify.com` domain |
| `ADMIN_API_ACCESS_TOKEN` | Shopify Admin API token for that store |
| `APP_ACCESS_TOKEN` | Swym's AppAccessToken for the merchant |
| `PID` | Swym's merchant/store identifier |

---

## 02 · CSV formats

### Combined registry CSV

Template: [Make a copy of the Google Sheet](https://docs.google.com/spreadsheets/d/1PKOby9EANuv248UBsltXk09wuUp5K3GhwI-5ZqGNjV8). Fill it in, then **File → Download → CSV**.

| Column | Required | Notes |
|---|---|---|
| `RegistryId` | Yes | Source registry id. Must match the item file name (`<RegistryId>.csv`). |
| `Email` | Yes | Registry owner's email. Rows without it are skipped. |
| `RegistryName` | Yes | |
| `Pid` | Yes* | Per-row Swym PID. *If empty, the script uses `PID` from `.env`. |
| `ExpiryDate` | No | Any date `new Date()` can parse. Sent as `YYYY-MM-DD`. |
| `Occasion` | No | Empty or `N/A` becomes `Others`. |
| `Mode` | No | Defaults to `public`. |
| `Settings → IsPasswordProtected` | No | `true` / `false`. If `true`, `Password` is sent. |
| `IsDeleted`, `IsArchived` | No | `true` rows are skipped (see `SKIP_DELETED`, `SKIP_ARCHIVED`). |
| `CustomProps` | No | JSON string. Kept as `sourceCustomProps`. |
| `Address → *` | No | Mapped into the registry address. |

### Individual registry item CSV (`<RegistryId>.csv`)

Template: [Make a copy of the Google Sheet](https://docs.google.com/spreadsheets/d/1PKOby9EANuv248UBsltXk09wuUp5K3GhwI-5ZqGNjV8/edit). Fill it in, then **File → Download → CSV** and name the file `<RegistryId>.csv`.

| # | Column | Required | Notes |
|---|---|---|---|
| 1 | `product id(empi)` | Yes | Shopify product id. Rows without it are skipped. |
| 2 | `variant id (epi)` | Yes | Shopify variant id. Rows without it are skipped. |
| 3 | `Handle` | No | Product handle or full URL. Builds the product link. |
| 4 | `Pid` | No | Not used for API calls (the registry row's `Pid` is used). |
| 5 | `RegistryId` | No | A warning is logged if it does not match the file name. |
| 6 | `AskQuantity` | No | Empty, `0` or invalid becomes `1`. |
| 7 | `CustomProps` | No | Not used. |
| 8 | `BoughtQuantity` | No | If `1` or more, the item is marked as bought. |
| 9 | `CreatedAt` | No | Not used. |
| 10 | `UpdatedAt` | No | Not used. |

---

## 03 · `.env` format

```bash
# Shopify store (target store you are importing INTO)
SHOP=your-store.myshopify.com
SHOPIFY_API_VERSION=2026-04
ADMIN_API_ACCESS_TOKEN=shpat_xxxxxxxxxxxxxxxxxxxxxxxxxxxx

# Swym Gift Registry
APP_ACCESS_TOKEN=your-swym-app-access-token
SWYM_REGISTRY_API_URL=https://api.swymregistry.com/giftRegistry/v1

# Fallback Pid, only used if a registry row has no Pid of its own
PID=

# Input files (paths are relative to the script's folder)
COMBINED_REGISTRY_CSV=../<combined-registries>.csv
INDIVIDUAL_REGISTRY_DIR=..

# Optional
SKIP_DELETED=true
SKIP_ARCHIVED=true
IMPORT_LIMIT=
```

---

## 04 · Script

One script handles both the combined registry file and its per-registry item files. It reads all config from `.env`.

<details>
<summary><b>▸ Show script</b></summary>

```js
const fs = require("fs");
const path = require("path");
const csv = require("@fast-csv/parse");

// ---------------------------------------------------------------------------
// Imports the La Coqueta UK dataset (La_Coqueta_UK_All_registries.csv + one
// <RegistryId>.csv per registry) into Swym's Gift Registry. Adapted from
// ../../gift-registry-import/src/importLacoquetaRegistry.js and
// ../../gift-reg-test-import/src/importAuraRegistry.js:
//  - registry id column is "RegistryId", not "ID"
//  - item rows already carry "product id(empi)" directly, so no Shopify
//    variant lookup is needed to resolve a product id
//  - each row carries its own "Pid" - that per-row Pid is used for every Swym
//    API call instead of a single global PID (per user's explicit choice)
//  - item files here ship with a corrupted header row (tab-joined names
//    followed by a stray duplicated comma-joined tail), so item columns are
//    assigned by fixed position instead of trusting the file's own header
//
// Run `npm install` in this folder first (installs @fast-csv/parse).
// Config comes from environment variables - see .env.example.
// ---------------------------------------------------------------------------

const DRY_RUN = process.argv.includes("--dry-run") || parseBool(process.env.DRY_RUN);

const config = {
  shop: requireEnv("SHOP"),
  shopifyApiVersion: process.env.SHOPIFY_API_VERSION || "2026-04",
  adminApiAccessToken: requireEnv("ADMIN_API_ACCESS_TOKEN"),
  appAccessToken: requireEnv("APP_ACCESS_TOKEN"),
  fallbackPid: process.env.PID || null,
  swymRegistryApiUrl:
    process.env.SWYM_REGISTRY_API_URL ||
    "https://api.swymregistry.com/giftRegistry/v1",
  combinedRegistryCsv: path.resolve(
    __dirname,
    process.env.COMBINED_REGISTRY_CSV || "../La_Coqueta_UK_All_registries.csv"
  ),
  individualRegistryDir: path.resolve(
    __dirname,
    process.env.INDIVIDUAL_REGISTRY_DIR || ".."
  ),
  skipDeleted: process.env.SKIP_DELETED === undefined ? true : parseBool(process.env.SKIP_DELETED),
  skipArchived: process.env.SKIP_ARCHIVED === undefined ? true : parseBool(process.env.SKIP_ARCHIVED),
  importLimit: process.env.IMPORT_LIMIT ? parseInt(process.env.IMPORT_LIMIT, 10) : null,
};

function requireEnv(name) {
  const value = process.env[name];
  if (!value) {
    console.error(`Missing required environment variable: ${name}`);
    console.error("Copy .env.example to .env and fill it in, then run with: node --env-file=.env src/importRegistry.js");
    process.exit(1);
  }
  return value;
}

function parseBool(value) {
  return String(value).trim().toLowerCase() === "true";
}

// ---------------------------------------------------------------------------
// Column mapping - matches La_Coqueta_UK_All_registries.csv / <RegistryId>.csv headers.
// ---------------------------------------------------------------------------

const REGISTRY_COLUMNS = {
  id: "RegistryId",
  pid: "Pid",
  name: "RegistryName",
  creatorName: "CreatorName",
  coCreatorName: "CoCreatorName",
  email: "Email",
  description: "Description",
  expiryDate: "ExpiryDate",
  occasion: "Occasion",
  mode: "Mode",
  password: "Password",
  isPasswordProtected: "Settings → IsPasswordProtected",
  isDeleted: "IsDeleted",
  isArchived: "IsArchived",
  addressFirstName: "Address → FirstName",
  addressLastName: "Address → LastName",
  addressAddress1: "Address → Address1",
  addressAddress2: "Address → Address2",
  addressCity: "Address → City",
  addressProvince: "Address → Province",
  addressCountry: "Address → Country",
  addressZip: "Address → Zip",
  addressPhone: "Address → Phone",
  addressCompany: "Address → Company",
  addressStreet: "Address → Street",
};

// Columns already accounted for above, plus other known source columns that
// should NOT be re-added into customProps as leftovers.
const KNOWN_REGISTRY_COLUMNS = new Set([
  ...Object.values(REGISTRY_COLUMNS),
  "CreatedAt",
  "UpdatedAt",
  "AssociatedListId",
  "Address",
  "Settings",
  "CustomProps",
  "ImageURL",
  "ArchivalDate",
]);

const ITEM_COLUMNS = {
  productId: "product id(empi)",
  variantId: "variant id (epi)",
  handle: "Handle",
  registryId: "RegistryId",
  askQuantity: "AskQuantity",
  boughtQuantity: "BoughtQuantity",
};

// Fixed column order for per-registry item files. This dataset's item CSVs
// ship with a corrupted header row - tab-joined names followed by a stray
// duplicated comma-joined tail - even though the data rows underneath are
// clean, correctly-ordered comma CSV. Rather than trust each file's own
// (broken) header row, we discard it and assign this known order explicitly.
const ITEM_FILE_COLUMN_ORDER = [
  "product id(empi)",
  "variant id (epi)",
  "Handle",
  "Pid",
  "RegistryId",
  "AskQuantity",
  "CustomProps",
  "BoughtQuantity",
  "CreatedAt",
  "UpdatedAt",
];

// ---------------------------------------------------------------------------
// Helpers
// ---------------------------------------------------------------------------

const formatDate = function (dateString) {
  const date = new Date(dateString);
  if (isNaN(date.getTime())) return null;
  const year = date.getFullYear();
  const month = String(date.getMonth() + 1).padStart(2, "0");
  const day = String(date.getDate()).padStart(2, "0");
  return `${year}-${month}-${day}`;
};

// A missing file's ENOENT surfaces on the underlying ReadStream, not on the
// csv parser's own "error" event - fast-csv doesn't forward it, so it would
// otherwise crash the process as an unhandled error instead of rejecting.
// Checking existence first avoids relying on that error propagation at all.
const getFileDetails = function (filePath) {
  return new Promise((resolve, reject) => {
    if (!fs.existsSync(filePath)) {
      reject(new Error(`File not found: ${filePath}`));
      return;
    }
    const data = [];
    csv
      .parseFile(filePath, { headers: (headers) => headers.map((h) => h.trim()) })
      .on("error", (error) => reject(error))
      .on("data", (row) => data.push(row))
      .on("end", () => resolve(data));
  });
};

// Item files: ignore whatever the file's own header row says (see
// ITEM_FILE_COLUMN_ORDER above) and assign column names by fixed position.
const getItemFileDetails = function (filePath) {
  return new Promise((resolve, reject) => {
    if (!fs.existsSync(filePath)) {
      reject(new Error(`File not found: ${filePath}`));
      return;
    }
    const data = [];
    csv
      .parseFile(filePath, { headers: ITEM_FILE_COLUMN_ORDER, renameHeaders: true })
      .on("error", (error) => reject(error))
      .on("data", (row) => data.push(row))
      .on("end", () => resolve(data));
  });
};

function tryParseJson(value) {
  if (!value) return null;
  try {
    return JSON.parse(value);
  } catch {
    return null;
  }
}

function resolveDestinationUrl(handle) {
  if (!handle) return `https://${config.shop}/products/`;
  return handle.startsWith("http") ? handle : `https://${config.shop}/products/${handle}`;
}

// ---------------------------------------------------------------------------
// Shopify Admin API
// ---------------------------------------------------------------------------

async function shopifyGet(urlPath) {
  const response = await fetch(
    `https://${config.shop}/admin/api/${config.shopifyApiVersion}${urlPath}`,
    { headers: { "X-Shopify-Access-Token": config.adminApiAccessToken } }
  );
  if (!response.ok) {
    throw new Error(`Shopify GET ${urlPath} failed: ${response.status} ${await response.text()}`);
  }
  return response.json();
}

async function findCustomerByEmail(email) {
  const data = await shopifyGet(`/customers/search.json?query=${encodeURIComponent(`email:${email}`)}`);
  return data.customers && data.customers.length ? data.customers[0] : null;
}

async function createCustomer(email) {
  const response = await fetch(
    `https://${config.shop}/admin/api/${config.shopifyApiVersion}/customers.json`,
    {
      method: "POST",
      headers: {
        "X-Shopify-Access-Token": config.adminApiAccessToken,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        customer: { send_email_invite: true, email, verified_email: true },
      }),
    }
  );
  if (!response.ok) {
    throw new Error(`Customer create failed for ${email}: ${response.status} ${await response.text()}`);
  }
  const data = await response.json();
  return data.customer;
}

// ---------------------------------------------------------------------------
// Swym Gift Registry API - all calls take an explicit pid (the per-row Pid
// from the CSV), rather than a single global config.pid.
// ---------------------------------------------------------------------------

async function getJwt(email, pid, { allowCustomerCreate }) {
  let customer = await findCustomerByEmail(email);
  let platformCustomerId;

  if (customer) {
    platformCustomerId = email;
  } else if (allowCustomerCreate) {
    const newCustomer = await createCustomer(email);
    platformCustomerId = String(newCustomer.id);
  } else {
    return { jwt: null, customerExists: false };
  }

  const response = await fetch(
    `${config.swymRegistryApiUrl}/shopper/generateAuth?pid=${encodeURIComponent(pid)}`,
    {
      method: "POST",
      headers: {
        accept: "application/json",
        SWYM_MERCHANT_API_KEY: config.appAccessToken,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        email,
        appId: "29d24356-b3d3-4c6e-9efe-68209e24687d",
        platformCustomerId,
      }),
    }
  );
  const data = await response.json();
  return { jwt: data.token, customerExists: !!customer };
}

async function findExistingRegistry(email, registryName, expiryDate, pid, jwt) {
  const response = await fetch(
    `${config.swymRegistryApiUrl}/shopperPrime/registry?userEmail=${encodeURIComponent(email)}&pid=${encodeURIComponent(pid)}`,
    { headers: { accept: "application/json", jwtToken: jwt, "Content-Type": "application/json" } }
  ).then((res) => (res.status === 200 ? res.json() : null));

  return response?.find(
    (registry) =>
      registry.registryName === registryName &&
      registry.email === email &&
      registry.expiryDate === expiryDate
  );
}

async function createRegistry(jwt, payload) {
  const response = await fetch(`${config.swymRegistryApiUrl}/shopperPrime/registry`, {
    method: "POST",
    headers: { accept: "application/json", jwtToken: jwt, "Content-Type": "application/json" },
    body: JSON.stringify(payload),
  });
  if (!response.ok) console.log(await response.text());
  return response.json();
}

async function addToRegistry(registryId, jwt, payload) {
  const reqUrl = new URL(`${config.swymRegistryApiUrl}/shopperPrime/registry/modifyProducts`);
  reqUrl.searchParams.append("registryId", registryId);
  const response = await fetch(reqUrl, {
    method: "POST",
    headers: { Accept: "application/json", "Content-Type": "application/json", jwtToken: jwt },
    body: JSON.stringify(payload),
  });
  if (!response.ok) console.log(await response.text());
  return response.json();
}

async function updateProductQuantity(jwt, registryId, payload) {
  const response = await fetch(
    `${config.swymRegistryApiUrl}/shopperPrime/registry/productQuantity?registryId=${registryId}`,
    {
      method: "PUT",
      headers: { accept: "application/json", jwtToken: jwt, "Content-Type": "application/json" },
      body: JSON.stringify(payload),
    }
  );
  if (!response.ok) console.log(await response.text());
  return response.json();
}

async function markBoughtProducts(registryId, pid, products) {
  const promises = products.map(async (item) => {
    if (item.quantity < 1 || isNaN(item.quantity)) return;
    const url = `${config.swymRegistryApiUrl}/storefront/markBought?registryId=${registryId}&pid=${encodeURIComponent(pid)}`;
    const response = await fetch(url, {
      method: "POST",
      headers: { accept: "application/json", "Content-Type": "application/json" },
      body: JSON.stringify([{ epi: `${item.variantId}`, empi: `${item.empi}`, qty: item.quantity }]),
    });
    if (!response.ok) console.log(`markBought failed for variant ${item.variantId}:`, await response.text());
  });
  await Promise.all(promises);
}

// ---------------------------------------------------------------------------
// Row -> payload mapping
// ---------------------------------------------------------------------------

function buildRegistryPayload(row) {
  const isPasswordProtected = parseBool(row[REGISTRY_COLUMNS.isPasswordProtected]);
  const expiryDate = formatDate(row[REGISTRY_COLUMNS.expiryDate]);

  const payload = {
    registryName: row[REGISTRY_COLUMNS.name],
    creatorName: row[REGISTRY_COLUMNS.creatorName] || "",
    coCreatorName: row[REGISTRY_COLUMNS.coCreatorName] || "",
    email: row[REGISTRY_COLUMNS.email],
    description: row[REGISTRY_COLUMNS.description] || "",
    expiryDate,
    customProps: [],
    mode: row[REGISTRY_COLUMNS.mode] || "public",
    settings: { isPasswordProtected },
    address: {
      firstName: row[REGISTRY_COLUMNS.addressFirstName] || "",
      lastName: row[REGISTRY_COLUMNS.addressLastName] || "",
      address1: row[REGISTRY_COLUMNS.addressAddress1] || "",
      address2: row[REGISTRY_COLUMNS.addressAddress2] || "",
      city: row[REGISTRY_COLUMNS.addressCity] || "",
      province: row[REGISTRY_COLUMNS.addressProvince] || "",
      zip: row[REGISTRY_COLUMNS.addressZip] || "",
      country: row[REGISTRY_COLUMNS.addressCountry] || "",
      phone: row[REGISTRY_COLUMNS.addressPhone] || "",
      company: row[REGISTRY_COLUMNS.addressCompany] || "",
      street: row[REGISTRY_COLUMNS.addressStreet] || "",
    },
    occasion:
      !row[REGISTRY_COLUMNS.occasion] || row[REGISTRY_COLUMNS.occasion] === "N/A"
        ? "Others"
        : row[REGISTRY_COLUMNS.occasion],
  };

  if (isPasswordProtected && row[REGISTRY_COLUMNS.password]) {
    payload.settings.password = row[REGISTRY_COLUMNS.password];
  }

  const customProp = {};
  for (const key of Object.keys(row)) {
    if (!KNOWN_REGISTRY_COLUMNS.has(key)) customProp[key] = row[key];
  }
  const parsedCustomProps = tryParseJson(row.CustomProps);
  if (parsedCustomProps) customProp.sourceCustomProps = parsedCustomProps;
  payload.customProps.push(customProp);

  return payload;
}

function buildItemPayloads(itemRows, registryId) {
  const addPayload = { add: [] };
  const quantityPayload = [];
  const boughtProducts = [];

  for (const item of itemRows) {
    const variantId = item[ITEM_COLUMNS.variantId];
    const productId = item[ITEM_COLUMNS.productId];
    const handle = item[ITEM_COLUMNS.handle];
    const rowRegistryId = item[ITEM_COLUMNS.registryId];

    if (rowRegistryId && String(rowRegistryId) !== String(registryId)) {
      console.log(`Warning: item row RegistryId (${rowRegistryId}) does not match file's registry id (${registryId})`);
    }

    if (!variantId || !productId) {
      console.log(`Skipping item with missing VariantId/ProductId in registry ${registryId}:`, item);
      continue;
    }

    addPayload.add.push({
      epi: variantId,
      empi: productId,
      du: resolveDestinationUrl(handle),
    });

    const askQuantity = parseInt(item[ITEM_COLUMNS.askQuantity], 10);
    quantityPayload.push({ epi: variantId, askQuantity: askQuantity <= 0 || isNaN(askQuantity) ? 1 : askQuantity });

    boughtProducts.push({
      empi: productId,
      variantId,
      quantity: parseInt(item[ITEM_COLUMNS.boughtQuantity], 10),
      debug: item,
    });
  }

  return { addPayload, quantityPayload, boughtProducts };
}

// ---------------------------------------------------------------------------
// Main
// ---------------------------------------------------------------------------

const results = {
  success: [],
  failed: [],
  skipped: [],
  noItemsFile: [],
};

async function processRegistry(row, index, total) {
  const registryId = row[REGISTRY_COLUMNS.id];
  const email = row[REGISTRY_COLUMNS.email];
  const pid = row[REGISTRY_COLUMNS.pid] || config.fallbackPid;
  const label = `[${index + 1}/${total}] registry ${registryId}`;

  if (config.skipDeleted && parseBool(row[REGISTRY_COLUMNS.isDeleted])) {
    console.log(`${label}: skipped (IsDeleted)`);
    results.skipped.push({ registryId, reason: "deleted" });
    return;
  }
  if (config.skipArchived && parseBool(row[REGISTRY_COLUMNS.isArchived])) {
    console.log(`${label}: skipped (IsArchived)`);
    results.skipped.push({ registryId, reason: "archived" });
    return;
  }
  if (!email) {
    console.log(`${label}: skipped (no Email)`);
    results.skipped.push({ registryId, reason: "no email" });
    return;
  }
  if (!pid) {
    console.log(`${label}: skipped (no Pid on row and no PID fallback configured)`);
    results.skipped.push({ registryId, reason: "no pid" });
    return;
  }

  const payload = buildRegistryPayload(row);

  let itemRows;
  const itemsFilePath = path.join(config.individualRegistryDir, `${registryId}.csv`);
  try {
    itemRows = await getItemFileDetails(itemsFilePath);
  } catch (error) {
    console.log(`${label}: no items file found at ${itemsFilePath}`);
    results.noItemsFile.push({ registryId });
    itemRows = [];
  }

  if (DRY_RUN) {
    let customer = null;
    try {
      customer = await findCustomerByEmail(email);
    } catch (error) {
      console.log(`${label}: customer lookup failed - ${error.message}`);
    }

    let existingRegistry = null;
    if (customer) {
      try {
        const { jwt } = await getJwt(email, pid, { allowCustomerCreate: false });
        if (jwt) existingRegistry = await findExistingRegistry(email, payload.registryName, payload.expiryDate, pid, jwt);
      } catch (error) {
        console.log(`${label}: auth/lookup check failed - ${error.message}`);
      }
    }

    const { addPayload } = buildItemPayloads(itemRows, registryId);

    console.log(
      `${label}: DRY RUN - pid ${pid}, customer ${customer ? "exists" : "would be created"}, ` +
        `registry ${existingRegistry ? "already exists (would reuse)" : "would be created"}, ` +
        `${addPayload.add.length}/${itemRows.length} items have a usable ProductId/VariantId`
    );
    results.success.push({ registryId, dryRun: true, payload, itemCount: itemRows.length, resolvedCount: addPayload.add.length });
    return;
  }

  try {
    const { jwt, customerExists } = await getJwt(email, pid, { allowCustomerCreate: true });
    if (!jwt) throw new Error("Failed to obtain JWT");

    const existing = await findExistingRegistry(email, payload.registryName, payload.expiryDate, pid, jwt);
    const registryDetails = existing || (await createRegistry(jwt, payload));

    if (!registryDetails || !registryDetails.Id) {
      throw new Error(`Registry create/lookup did not return an Id (customerExists=${customerExists})`);
    }

    const { addPayload, quantityPayload, boughtProducts } = buildItemPayloads(itemRows, registryId);

    if (addPayload.add.length) {
      await addToRegistry(registryDetails.Id, jwt, addPayload);
      await updateProductQuantity(jwt, registryDetails.Id, quantityPayload);
      await markBoughtProducts(registryDetails.Id, pid, boughtProducts);
    }

    console.log(`${label}: imported successfully as registry ${registryDetails.Id}`);
    results.success.push({ registryId, swymRegistryId: registryDetails.Id, itemCount: addPayload.add.length });
  } catch (error) {
    console.log(`${label}: FAILED - ${error.message}`);
    results.failed.push({ registryId, error: error.message });
  }
}

async function main() {
  console.log(`Mode: ${DRY_RUN ? "DRY RUN (no data will be written)" : "LIVE IMPORT"}`);
  console.log(`Combined registry CSV: ${config.combinedRegistryCsv}`);
  console.log(`Individual registry dir: ${config.individualRegistryDir}`);

  const registries = await getFileDetails(config.combinedRegistryCsv);
  const toProcess = config.importLimit ? registries.slice(0, config.importLimit) : registries;
  console.log(`Loaded ${registries.length} registries${config.importLimit ? `, processing first ${toProcess.length}` : ""}`);

  for (let i = 0; i < toProcess.length; i++) {
    await processRegistry(toProcess[i], i, toProcess.length);
  }

  const outFile = path.resolve(
    __dirname,
    "..",
    DRY_RUN ? "dry_run_report.json" : "import_results.json"
  );
  fs.writeFileSync(outFile, JSON.stringify(results, null, 2));

  console.log("\n--- Summary ---");
  console.log(`Success: ${results.success.length}`);
  console.log(`Failed: ${results.failed.length}`);
  console.log(`Skipped: ${results.skipped.length}`);
  console.log(`No items file found: ${results.noItemsFile.length}`);
  console.log(`Details written to: ${outFile}`);
}

if (require.main === module) {
  main().catch((error) => {
    console.error("Fatal error:", error);
    process.exit(1);
  });
}

module.exports = { main, buildRegistryPayload, buildItemPayloads, formatDate };
```

</details>

---

## 05 · How to run

```bash
cp .env.example .env      # fill in SHOP, ADMIN_API_ACCESS_TOKEN, APP_ACCESS_TOKEN, PID
npm install
npm run import:dry-run    # checks only, no writes
npm run import            # live import
```
