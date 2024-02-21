# dbt project structure

Dalgo can manage the `dbt` repos for multiple tenants. Each `Org` who has a `dbt` configuration has an `OrgDbt` configuration:

```
OrgDbt
    gitrepo_url
    gitrepo_access_token_secret

    project_dir
    dbt_venv

    target_type
    default_schema
```
(Currently each org can only have one dbt project configured)

The `gitrepo_url` is the full URL to their GitHub repo. If the repo is private, then Dalgo will need to pull from it with a [Personal Access Token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens), and we store their `gitrepo_access_token_secret` for this purpose.

The `project_dir` points to the location on the disk where the Dalgo backend is running, i.e. `/home/ubuntu/...`. 

Under a single folder `CLIENTDBT_ROOT` defined in `.env`, we create a folder named for the `Org` using their `org.slug`. Within this folder we clone thier GitHub repo into a folder called `dbtrepo/`.

So for example:
- if there is an `Org` having as its slug the string `myorg`
- and if `CLIENTDBT_ROOT` is set to `/home/ubuntu/clientdbt_root/`

, then the client's repo is cloned into `/home/ubuntu/clientdbt_root/myorg/dbtrepo/`

The `project_dir` is set to `/home/ubuntu/clientdbt_root/myorg/`, and so the dbt repo is under `<project_dir>/dbtrepo/`.

The `dbt_venv` points to the Python virtual environment where `dbt` is installed. By default a new tenant's `dbt_venv` is set to the global `DBT_VENV` defined in `.env`, but this could be overridden if required.

The `target_type` is the warehouse type e.g. `"postgres"` or `"bigquery"`. 

Finally, the `default_schema` is the warehouse schema/dataset which `dbt` will write to by default.
