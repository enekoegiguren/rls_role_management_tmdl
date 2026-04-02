# fabric-rls-role-management

Automate the creation, replacement, and member assignment of static Row Level Security (RLS) roles in a Power BI Semantic Model using a Microsoft Fabric notebook.

Built on [`semantic-link-labs`](https://github.com/microsoft/semantic-link-labs) and the Tabular Object Model (TOM), this notebook replaces the need for manual role management or Dynamic RLS security tables — both of which become painful at scale.

---

## The problem

When a Power BI model has complex, multi-dimensional security requirements — multiple business units, territories, countries, brands — the typical solution is **Dynamic RLS**: a security table in the model filtered by `USERPRINCIPALNAME()` at query time.

This works. Until it doesn't.

On a fact table with 10M+ rows and a security table with hundreds of combinations, Dynamic RLS evaluates on **every query, for every user**. Reports slow down. Users complain. And asking your Azure AD admin to manually create 500 security groups is not a conversation anyone wants to have.

**Static RLS roles are evaluated once at connection time** — VertiPaq resolves a fixed DAX expression instantly and propagates it through relationships. The bottleneck disappears.

The problem has always been: who creates 500 static roles? This notebook does.

---

## How it works

```
RLS table (Lakehouse)
        │
        ▼
Spark DataFrame (rls)
        │
        ├─ build_dax_filters()      → DAX expression per role per table
        ├─ create_or_replace_roles() → push roles + filters to semantic model
        └─ add_members_to_roles()   → assign UPN emails to each role
                                           │
                                           ▼
                              Published Semantic Model
```

1. You define a `config` — one entry per dimension to secure (e.g. `CountryCode`, `TerritoryCode`, `BrandingCode`)
2. You define `global_filters` — filters applied to every role on their own fixed table (e.g. `Is_Consolidated = "Consolidated"` on `DT_Customer`)
3. The notebook reads distinct values from your RLS DataFrame, generates a role per value, applies the DAX filters, and assigns members from the `Username` column

---

## Design principles

### Filter on dimension tables (DT), not fact tables (FT)

Filtering directly on a fact table with millions of rows means the engine scans it on every query for every user. Filtering on a dimension table lets the relationship engine propagate the filter automatically — which is exactly what VertiPaq is optimized for.

```
DT_Customer [CountryCode = "AU"]  →  propagates through relationship  →  FT_Sales
```

### Static over dynamic

| | Dynamic RLS | Static RLS (this notebook) |
|---|---|---|
| Evaluated | Every query, every user | Once at connection time |
| Security table in model | Yes — adds model complexity | No |
| Maintenance | Manual or scripted | Scripted, re-runnable |
| Performance on 10M+ rows | Slow | Fast |
| 500 combinations | Very slow | Fine |

### Global filters are always separate

`global_filters` are applied to their own fixed table regardless of what the role-specific filter uses. If both land on the same table, they are merged with `&&` into a single `set_rls` call — because `set_rls` replaces, not appends.

### Members are saved one at a time

A single invalid UPN causes `SaveChanges()` to reject the entire batch. By opening a fresh TOM connection per member, a bad UPN never blocks a valid one. Failed members are collected and exported as a JSON report to your Lakehouse `Files` folder.

---

## Requirements

- Microsoft Fabric workspace with a Lakehouse attached to the notebook
- Power BI Semantic Model published to the workspace
- XMLA Read/Write enabled on the Fabric capacity
- `semantic-link-labs` (installed in the notebook via `%pip install`)
- RLS source table with at minimum: `Username` (UPN email) + dimension columns

---

## Configuration

### `global_filters`

Filters applied to **every** role, each on their own fixed table.

```python
global_filters = [
    {
        "table":  "DT_Customer",    # Table in the semantic model
        "column": "Is_Consolidated",
        "value":  "Consolidated"
    },
    # Add as many as needed
]
```

### `config`

One entry per dimension. The key must match a column name in your RLS DataFrame.

```python
config = {
    "CountryCode": {
        "table":  "DT_Customer",        # Dimension table — not the fact table
        "prefix": "RLS_CountryCode",    # Roles will be named RLS_CountryCode_AU, etc.
        "column": "CountryCode"         # Column in the semantic model table
    },
    "BrandingCode": {
        "table":  "DT_Product",
        "prefix": "RLS_BrandingCode",
        "column": "BrandingCode"
    },
}
```

### RLS DataFrame (`rls`)

Must contain:

| Column | Description |
|--------|-------------|
| `Username` | UPN email (`user@domain.com`) |
| `CountryCode` | One column per `config` key |
| `BrandingCode` | ... |

---

## Usage

```python
# Step 1 — always safe to re-run (idempotent)
create_or_replace_roles(
    config         = config,
    dataset        = dataset,
    workspace      = workspace,
    global_filters = global_filters,
    rls            = rls,
    config_keys    = None,   # None = all; or ["CountryCode"] to run a subset
)

# Step 2 — optional, run separately once roles are confirmed
failed = add_members_to_roles(
    config       = config,
    dataset      = dataset,
    workspace    = workspace,
    rls          = rls,
    username_col = "Username",
    config_keys  = None,
)
```

Both functions accept a `config_keys` parameter to process only a subset of the config — useful when iterating or debugging a specific dimension.

---

## Output

- Roles created in the semantic model: `RLS_CountryCode_AU`, `RLS_CountryCode_MT`, ...
- Each role has the DAX filter applied per table
- Members assigned from the `Username` column in the RLS DataFrame
- Failed members exported to `Files/rls_failed_members/` in the Lakehouse as JSON

---

## File structure

```
fabric-rls-role-management/
├── RLS_Role_Management.ipynb   ← main notebook
└── README.md
```

---

## Related

- [semantic-link-labs](https://github.com/microsoft/semantic-link-labs) — the library powering the TOM connection
- [TMDL view in Power BI Desktop](https://learn.microsoft.com/en-us/power-bi/transform-model/desktop-tmdl-view) — the broader TMDL ecosystem this fits into
- [Semantic model connectivity with XMLA endpoint](https://learn.microsoft.com/en-us/power-bi/enterprise/service-premium-connect-tools)

---

## Contributing

Issues and PRs welcome. If your security model has patterns not covered by the current `config` structure — multiple fact table filters, OLS, cross-workspace models — open an issue and let's discuss.

---

## License

MIT
