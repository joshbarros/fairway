# Adding a corridor

A corridor is one sell asset and one buy asset that Fairway prices through Stellar Horizon. Adding one takes four steps: identify the assets, verify both issuers, confirm the FX mapping, and add the entry to `corridors.yaml`.

Use the existing entries in `corridors.yaml` and the evidence notes in [`corridor-verification.md`](corridor-verification.md) as examples. Do not mark an issuer `live` unless you can point to the evidence that supports it.

## 1. Identify both assets

You need the asset code and the issuer account for each side.

- Native XLM has no issuer. Use an empty string for its issuer field. XLM is not currently a supported corridor side for FX benchmarking; see step 3.
- For issued assets, find the issuer on Stellar Expert or query Horizon:

```bash
curl --get 'https://horizon.stellar.org/assets' \
  --data-urlencode 'asset_code=USDC' \
  --data-urlencode 'asset_issuer=GA5ZSEJYB37JRC5AVCIA5MOP4RHTM335X2KGX3IHOJAPP5RE34K4KZVN'
```

Confirm that the code and issuer pair is the asset you intend to measure. A code alone is not unique on Stellar.

## 2. Verify each issuer

Verification is per leg. Do it independently for the sell asset and the buy asset.

1. Fetch the issuer account and read `home_domain`:

```bash
curl 'https://horizon.stellar.org/accounts/<ISSUER_ACCOUNT_ID>'
```

2. If `home_domain` is set, fetch its SEP-1 metadata:

```bash
curl 'https://<HOME_DOMAIN>/.well-known/stellar.toml'
```

3. Record what you actually found:
   - `sell_domain` / `buy_domain`: the issuer's `home_domain`, or an empty string when it has none.
   - `sell_anchor_metadata` / `buy_anchor_metadata`: `full` when a usable `stellar.toml` exists, otherwise `none`.
   - `sell_verified_status` / `buy_verified_status`: `live`, `pending`, `unverifiable`, or `unknown`.

SEP-1 does not require a single `status` value. If the file has no status field, do not invent one. Record `unknown` when the evidence does not establish a status, or `unverifiable` when there is no metadata at all. You may record `live` without a metadata status field when other recorded evidence supports it, such as sustained payment activity or a reserve attestation; explain that evidence in the PR or verification notes.

Also check the issuer's authorization flags and supply or payment activity. A reachable `stellar.toml` does not by itself prove that the asset is usable or safe.

## 3. Confirm the FX mapping

Fairway converts each asset to a fiat currency for the independent FX benchmark. The mapping lives in `AssetToCurrency` in `internal/reference/reference.go`.

- `NGNC` maps to `NGN`, `USDC` to `USD`, `EURC` to `EUR`, and so on.
- A four-letter code ending in `C` falls back to its first three letters. For example, `BRLC` maps to `BRL`.
- If neither rule is correct for your asset, add an explicit case and a unit test in `internal/reference/reference_test.go`.
- `AssetToCurrency` is keyed only by the asset code. If two issuers share a code with different fiat pegs, do not add a duplicate case; the asset is unsupported until the mapping can distinguish issuers.

Both sides need a mapping. If a side cannot be mapped to a fiat currency, the measurement cannot produce a meaningful USD-equivalent reference.

Native XLM is represented with an empty issuer, but `AssetToCurrency` returns `XLM`, which is not a fiat currency and has no reference-rate source in the current fetcher. Do not add an XLM corridor until a reference price source exists.

## 4. Add the configuration entry

Append the corridor to the file configured by `CORRIDORS_FILE` (`corridors.yaml` by default):

```yaml
  - name: "SELL/BUY"
    sell_asset_code:        "SELL"
    sell_asset_issuer:      "G..."
    buy_asset_code:         "BUY"
    buy_asset_issuer:       "G..."

    target_usd_value:       100.0

    sell_domain:            "sell.example"
    sell_anchor_metadata:   "full"
    sell_verified_status:   "live"

    buy_domain:             ""
    buy_anchor_metadata:    "none"
    buy_verified_status:    "unverifiable"

    verification_date: "YYYY-MM-DD"
    enabled:           true
```

Field notes:

- `name` must be unique and is the database key. Use `SELL/BUY` when it is unambiguous; if two issuers share the same asset code and fiat peg, add an issuer suffix (for example `SELL/BUY-GA...`) so the second entry does not overwrite the first.
- `target_usd_value` is the economic trade size. Fairway scales the sell amount with live reference rates instead of measuring a flat asset-unit amount.
- `sell_*` and `buy_*` describe the two legs independently.
- `verification_date` is the date you checked the metadata.
- `enabled: false` prevents a new entry from being seeded. For a corridor that is already in the database, set `enabled: false` in `corridors.yaml`, stop the app, disable the existing row, then rebuild and start the app with the disabled entry:

```bash
docker compose stop app
docker compose up -d --wait postgres
docker compose exec postgres psql -U fairway -d fairway -c "UPDATE corridors SET enabled = FALSE WHERE name = 'SELL/BUY' RETURNING name;"
docker compose build app
docker compose up -d --force-recreate app
```

Replace `SELL/BUY` with the corridor name. Confirm the update returned exactly one row before rebuilding; stop if it returns zero or multiple rows.

The loader skips disabled YAML entries. If the YAML entry stays `enabled: true`, the next startup re-enables the database row.

The same fields are stored in the `corridors` table in `schema.sql`; the YAML file is the source that Fairway seeds into PostgreSQL on startup.

## 5. Test locally

Create the local environment and start the stack:

```bash
[ -f .env ] || cp .env.example .env
docker compose up --build -d
```

These commands use the default `corridors.yaml`. If `CORRIDORS_FILE` points to a custom file, mount that file into the app container and set the same `CORRIDORS_FILE` value in `docker-compose.yml`; otherwise the custom file will not be seeded.

Wait for PostgreSQL and Fairway to start, then check health and the seeded corridors:

```bash
curl http://localhost:8080/healthz
curl http://localhost:8080/corridors
```

Watch the logs for the first measurement cycle:

```bash
docker compose logs -f app
```

A new corridor should appear in `/corridors`. After the scheduler runs, check the stored measurement, not just the `measured` log:

Take the corridor ID from `/corridors`, then substitute it in the URL:

```bash
# Example: corridor ID 1
curl http://localhost:8080/corridors/1/history
```

For the first row, confirm that `ErrorMsg` is `null`, `ReferenceRate` is present, and `SellAmount` is `target_usd_value` divided by the sell asset's USD rate. PostgreSQL stores `SellAmount` with seven decimal places, so compare after rounding to seven decimals.

`docker-compose.yml` sets `MEASURE_INTERVAL` to the literal `5m`, so changing `.env` alone does not shorten it. For a faster local loop, add a `docker-compose.override.yml`:

```yaml
services:
  app:
    environment:
      MEASURE_INTERVAL: 30s
```

Then rerun `docker compose up --build -d`.

If the corridor is disabled, confirm that it does not appear in the API response and is not measured.

## Before opening the PR

- Keep the diff limited to the corridor entry, any FX mapping, and the supporting evidence.
- Include the issuer identifiers and the verification evidence in the PR description or in `docs/corridor-verification.md`.
- Run `go vet ./...`, `golangci-lint run`, and the test suite against a reachable, disposable test database. When `DATABASE_URL` is unset, the store and scheduler tests may use default localhost databases on ports 5433 and 5432; they skip only when none responds. Do not rely on that fallback:

  ```bash
  # Use your own disposable test database, not the Compose postgres service.
  export DATABASE_URL='postgres://localhost:5432/fairway_test?sslmode=disable'
  psql "$DATABASE_URL" -f schema.sql
  go test -race -v ./...
  ```

  Use a credential-free disposable database, never a credential-bearing production URL; verbose test failures can print connection details. Check the output and confirm that the store and scheduler database tests actually ran. If the configured database is unreachable, the helper skips them and `go test` can still exit successfully.

- Do not include secrets, private credentials, or unverified claims about third-party APIs.
