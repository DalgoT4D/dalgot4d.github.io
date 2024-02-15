# Dalgo

## Introduction

Dalgo is an open-source data platform for the social sector. Dalgo is maintained by Project Tech4Dev which hosts a commercial multi-tenant instance at https://dashboard.dalgo.in/.

Dalgo tenants are called _organizations_. In our commercial offering, most organizations are client NGOs using our platform. We also have an organization for demos, and an organization for internal dashboarding. The platform does not distinguish between these in terms of functionality.

Every organization has an associated _warehouse_. We currently support Postgres and BigQuery warehouses, and could potentially support Snowflake and Redshift.

Dalgo allows an organization to configure multiple data sources and configure a frequency upon which to ingest data from those sources into the warehouse. We do this under the hood using [Airbyte](https://airbyte.com/).

Once the data is in the warehouse it typically undergoes several journeys to ready it for its various consumers. This is typically done using SQL and Dalgo supports this via [dbt](https://www.getdbt.com/).

Our support for different warehouse types is restricted only by what Airbyte and dbt support.

## Architecture

Dalgo uses [Prefect](https://www.prefect.io/) to manage job requirements across its tenants. This means that all Airbyte sync jobs and all dbt run jobs are queued into and executed by Prefect. Prefect then runs jobs on Airbyte and dbt and makes the logs available to the Dalgo platform.

<img src="https://github.com/DalgoT4D/dalgot4d.github.io/blob/main/dalgo-infrastructure.png" alt="Dalgo Infrastructure" />

The Dalgo backend is a [Django application](https://github.com/DalgoT4D/DDP_backend) which serves requests made from a [React frontend](https://github.com/DalgoT4D/webapp). The backend communicates with Prefect via a [lightweight proxy](https://github.com/DalgoT4D/prefect-proxy) since Prefect's Python SDK is all async and Django seems to support async Python only [half-heartedly](https://docs.djangoproject.com/en/5.0/topics/async/).

The backend maintains _organizations_ and _users_ in a database, and handles email communication to these users when required. Most users belong to only one organization, but the Dalgo team and our implementation partners are usually connected to more than one, and the platform lets us switch context under the same login. 

Every organization has its own dedicated Airbyte workspace, which acts as a container for its data sources and its data warehouse. Prefect has no corresponding support, so we attach metadata to Prefect configurations to help us track which organizations they belong to. The backend's database tracks every Prefect [_deployment_ ](https://docs.prefect.io/latest/concepts/deployments/) against its organization, and the platform retrieves _flow runs_ and associated logs through each deployment.

For Prefect to run dbt jobs, it requires dbt credentials to exist within a [dbt cli profile](https://prefecthq.github.io/prefect-dbt/#dbt-core-cli). The platform stores a reference to this profile for every organization.

Dalgo uses Prefect deployments in two ways:
1. For users to trigger an Airbyte sync or a dbt run from the UI
2. For Orchestration Pipelines, which can have a series of steps and usually run on a schedule

In general, we create a Prefect deployment for a task if we want to provide a history of logs for that task. We don't create deployments for
- `git pull`
- `dbt clean`
- `dbt deps`
- ...

Logs for these commands are displayed to the user when triggered from the UI, but are lost when they navigate away from the page.

## Users and Roles

We use Django's auth system and their standard [User](https://docs.djangoproject.com/en/5.0/topics/auth/default/#user-objects) model. Authentication is done via [`django-rest-framework`](https://www.django-rest-framework.org/) using [Token Authentication](https://www.django-rest-framework.org/api-guide/authentication/#tokenauthentication). Every email address maps to a unique `User` object.

Membership in an `Org` is tracked via an `OrgUser` object, which is essentially a `User`, an `Org` and a role. The platform currently has only three roles (with more to come):

1. `REPORT_VIEWER`
2. `PIPELINE_MANAGER`
3. `ACCOUNT_MANAGER`

All Django endpoints are decorated with one of the following role-checkers:
- `AnyOrgUser`: allows all three of the above
- `CanManagePipelines`: allows (2) and (3)
- `CanManageUsers`: allows (2) and (3)
- `FullAccess`: only (3)

You can see that the role system needs some work

## Celery

Some tasks which are triggered by http requests take too long to complete. We outsource these to [Celery](https://docs.celeryq.dev/en/latest/django/first-steps-with-django.html) which provides us a _task id_ using which we are able to poll the status of the task.  

Celery is also used to schedule regular maintenance. One example is to release `TaskLocks` which are not released during the normal cleanup proces.

