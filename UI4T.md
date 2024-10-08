# UI for Transformations

UI for Transformations ("UI4T") is a feature which allows a Dalgo user to create a transformation pipeline using a visual editor. The starting point is a list of tables in the user's warehouse, from which they select one and proceed to define a series of SQL transformations on it. The result of this is

- a dbt model on disk inside the organization's dbt repo, and
- a new table in the warehouse once the model is run

The transformation pipeline, or _workflow_ is represented visually using a directed graph. There are two node types:

- sources and models
- intermediate transformations

From a source or a model, the user can begin a sequence of transformations via a pop-up menu. Selecting a transformation type opens an off-canvas configuration form where the user specifies how the transformation should work. For example, for a `CAST` operation they will specify the column(s) to be cast and the new data types to which the values should be cast.

Once this operation has been specified, a new node will appear on the canvas which terminates the new edge originating from the model / source the user is transforming. The user can then either

- continue defining a sequence of transformations
- specify an end to the sequence and create a table

Creating a table results in the dbt model code being generated on disk, and once dbt is run successfully a new table will apepar in the warehouse.

New transformations can be created from 

- sources and models
- leaf transformations of a sequence under construction

## Implementation

Sources and models are tracked by organization in the `OrgDbtModel` table in the Dalgo database. These objects contain

- a UUID
- a path to the SQL file on disk
- a unique dbt model name
- a user-friendly display name for the UI
- the schema and table names in the warehouse
- the set of output columns from this model

We also maintain a table called `DbtEdge` which tracks the connections between sources and models. We do not track edges attached to intermediate transformations here.

Intermediate transformations are tracked in a separate table called `OrgDbtOperation`, and consist of

- a UUID
- the UUID of the model being defined
- the details of this transformation
- its position in the sequence
- the set of output columns from this operation

In order for us to do this, we need to create the `OrgDbtModel` of the model _still under construction_... we don't show it to the user on the design canvas but we do create it in our database. We distinguish these models via the flag `OrgDbtModel.is_materialized` which is set to `True` only once the model has been finalized by the user


## Editing

Editing an operation node means updating its configuration. We do not allow editing the _type_ of the operation i.e. a `drop` operation must remain a `drop` operation; only the columns being dropped can be changed.

Once a node is edited, the dbt sql on disc is updated and the input coliumns are updated for the next operation only. We do not propagete changes downstream. The user has to do this manually editing one operations after the other till they reach end of the chain. 

Editing at the source vs editing in the middle of chain differs in the fact that for the former case we also need to make sure the source edges (inputs) are preserved. This is done by making sure the `config->input_models` inside the `config` column for `OrgDbtOperation` is copied over & not overwrriten while updating.

Propagating changes downstream to the entire pipeline is more complicated. Let's take an example of a chain with two operations, drop followed by a replace:

   `src1 -> drop -> rename -> table1`
   
Assume the `drop` operation is configured to remove the columns `["col1", "col2"]`, and in the following `replace` operation we rename `"col3" -> "col3_new"`. 

Now if one edits the `drop` operation to drop column `col3`, the change cannot be propagated downstream, because `col3` no longer exists to be renamed.
