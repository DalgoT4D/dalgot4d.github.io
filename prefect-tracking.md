# Prefect and Dalgo

Commands which we run via Prefect are mapped to `Task`s. Tasks are fixed templates which are seeded from the `seed/tasks.json` file. At the time of this writing the only tasks are

- `git pull`
- Airbyte connection sync
- and the various `dbt` tasks

For every `Org` we create `OrgTask`s from these task templates. An `OrgTask` contains the command-line parameters for the task, if any. For the Airbyte sync task we store the Airbyte `connection_id` within the `OrgTask`.

Prefect deployments are tracked using the `OrgDataFlowv1` table. An `OrgDataFlowv1` essentially holds a `deployment_id` from Prefect and some scheduling information.

A Prefect deployment may consist of a series of tasks, and a single `OrgTask` may be a part of more than one deployment. This association is captured using the `DataflowOrgTask` table.

<img width="975" alt="image" src="https://github.com/DalgoT4D/dalgot4d.github.io/assets/2160416/0faf982a-f656-4b2d-8fde-7b3a0aeec34d">


## Creating dataflows

This is done in one of
- `orgtaskfunctions.create_prefect_deployment_for_dbtcore_task`
- `airbytehelpers.create_connection`
- `pipeline_api.post_prefect_dataflow_v1`

The first two create `manual` deployments i.e. deployments which don't show up in the Orchestrate page. All Orchestration deployments are created in the third function.

In each of these, we first create a Prefect deployment and then save the new `deployment_id` to the new `OrgDataFlowv1` object.


| Function | type | command | number of OrgTasks |
| -----------|------|---------|--------------------- |
| `orgtaskfunctions.create_prefect_deployment_for_dbtcore_task` | `manual` | `dbt run` | 1 |
| `airbytehelpers.create_connection` | `manual` | airbyte sync | 1 |
| `pipeline_api.post_prefect_dataflow_v1` | `orchestrate` | various | several |


## Deleting dataflows

Dataflows are deleted when we
1. Delete an Airbyte connection
2. Delete a `dbt` workspace
3. Delete an orchestration pipeline
