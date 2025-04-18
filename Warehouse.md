# The Warehouse

An Org sets up a warehouse in the frontend via `Ingest` `->` `Your Warehouse`. The form used is a mirror image of Airbyte's `Create-Destination` form, and all fields are sent to the Dalgo backend via `api/organizations/warehouse/`. 

At this time we create an Airbyte destination and save the `destinationId` to a new `OrgWarehouse` object. We also save the `dockerRepository` and `dockerImageTag` from Airbyte; these are currently only retrieved for display in `Ingest` `->` `Your Warehouse`.

Dalgo also needs access to warehouse configurations and credentials, for example for:
- Passing on to `dbt`
- Previewing tables in the warehouse

.. as well as multiple features on our roadmap

In a production system, these credentials are stored in AWS's Secrets Manager. Developers can save on their AWS costs by configuring `DEV_SECRETS_DIR` in `.env` to a location on disk; this will create a (highly insecure!) secrets manager locally.

## Storing warehouse credentials 
### Postgres
Airbyte sends over the following structure as of version `0.50.24`:

```
host, database, port, username, password
jdbc_url_params
ssl: true | false
ssl_mode:
  mode: disable | allow | prefer | require | verify-ca | verify-full
  ca_certificate: string if mode is require, verify-ca, or verify-full
  client_key_password: string if mode is verify-full
tunnel_method:
  tunnel_method = NO_TUNNEL | SSH_KEY_AUTH | SSH_PASSWORD_AUTH
  tunnel_host: string if SSH_KEY_AUTH | SSH_PASSWORD_AUTH
  tunnel_port: int if SSH_KEY_AUTH | SSH_PASSWORD_AUTH
  tunnel_user: string if SSH_KEY_AUTH | SSH_PASSWORD_AUTH
  ssh_key: string if SSH_KEY_AUTH
  tunnel_user_password: string if SSH_PASSWORD_AUTH
```

We store the object in our secrets manager as received


### BigQuery
For BigQuery we store only the GCP credentials JSON

# Retrieving warehouse credentials for use

As of this writing this is done in two places

## `dbt` CLI profiles
When a user sets up their `dbt` workspace via the `Transform` page in the frontend, we automatically call `api/prefect/tasks/transform/`. The credentials are retrieved from our secrets manager and passed on directly to `prefect-proxy`'s `proxy/blocks/dbtcli/profile/` endpoint, where it arrives in the function `_create_dbt_cli_profile`.

For Postgres, the credentials are stored in the org's `DbtCliProfile.target_configs.extras`. For BigQuery it goes into `DbtCliProfile.target_configs.credentials` after some formatting by Prefect's `prefect_gcp.GcpCredentials` block.

For both warehouse types, `DbtCliProfile.target_configs` is an important object. For Postgres it is a `TargetConfigs` instance, having `type="postgres"`, a specified `schema` and the `extras` dictionary.

For BigQuery it is a `BigQueryTargetConfigs` having `credentials`, `schema` and `{extras: {location}}`.


## The `warehouse` client

