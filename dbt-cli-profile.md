# dbt CLI Profile

The CLI profile is a [dbt object](https://prefecthq.github.io/prefect-dbt/#dbt-core-cli) which is used to populate a `profiles.yaml` under the dbt project directory. This file is read by dbt to fetch warehouse connection details (among other things). Every `DbtCoreOperation` must contain a link to a `DbtCliProfile` in order to run.

A `DbtCliProfile` contains
- a name
- a target
- a target config

The target config is a warehouse-specific configuration; for BigQuery it is a `BigQueryTargetConfigs` and for Postgres it is a `TargetConfigs`.

### `BigQueryTargetConfigs`

A `BigQueryTargetConfigs` is essentially
```
credentials: GcpCredentials(service_account_info=<service account JSON>)
schema: 
extras:
  location: BQ location i.e. data center
```

A `TargetConfigs` on the other hand, stores connection information under a key called `extras`:
```
type: "postgres"
schema:
extras:
  host:
  port:
  user:
  password:
  database:
  sslmode: disable | allow | prefer | require | ...
  sslrootcert: /path/to/root/cert
```

Dalgo stores a single `DbtCliProfile` block in Prefect. This profile can be viewed and modified using the Django command `addparamstodbtcliprofile`

- `python manage.py addparamstodbtcliprofile <org-slug>  --show`
- `python manage.py addparamstodbtcliprofile <org-slug>  --add <key> --value <value>`


