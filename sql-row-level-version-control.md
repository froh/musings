# sql row level version control
The common approach for row level versioning in sql rdms's is `valid_from` and `valid_thru` date fields in all tables under version control.
While this allows for a linar versioning it neither enables branching and merging nor is it simple to query;  for each table clauses are needed to check if the value is in the right date range.
I thought there has to be a better scheme to do row level versioning, with simpler queries and a branch/merge option.  This is what I came up with:

# versioning, ancestry dag, item trails

Versioning means having a directed acyclic graph (dag) of versions, the `anchestry-dag` (`ad`).

Each 'version' is a node in the anchestry-dag.  The edges of the graph, the `anchestry-dag-link`s, connect ancestor and successor.  Nodes with more than one successor are called branch points, nodes with more than one ancestor are called merge points.

Data items that are under such version control enter the system at some version, and they remain valid through a number of versions (possibly spanning branch and merge points).  The specific combination of versions for which the data item is the one to chose is the version trail of the data item in the anchestry dag:  it's a path in the dag.

For each version, each unique data point in each table can be in only one trail.

The trails can be identified (numbered/labeled) such that an otherwise identical data point will be on the 'same trail', i.e. have the same trail id accross all versions it's part of.

The set of all trails with data points in one version is the 'trail bundle' of that version in the ancestry dag.

# growing a twig:  Create, Update, Delete items

An insertion of new items in some version means adding a new trail and adding these new items to that new trail:  adding them to an existing trail would add them to other versions (which the existing trail is also part of).

Deleting an item in some version means shifting it to a new trail which is almost identical to the item's previous trail, with the difference that it is not part of this version.  Before a read to the older versions in the new trail, the new trail has to be added to all other versions that contained the item before deletion:  we've modified the item's trail, so we need to ensure it's still part of the other versions of that trail.

Modifying an item is deleting the previous version and inserting a new one.

# data model: meta tables and versioning info in data tables

all tables are prefixed by `ad` for ancestry dag

## versions, nodes in the ancestry dag

    ad-node
    - version-id
    - comment
    - date
    - work-in-progress

While a node is work in progress, no successor nodes must be added to this node, to prevent the time paradoxon (how to inherit genetic changes after birth?)

## edges of the ancestry dag

    ad-link
    - ancestor
    - successor

## trails

all data items live on some trail in the ancestry dag (this is the foreign key of all versioned data tables, the link into the version control:

    ad-trail
    - trail-id

each version is a collection of trails, each trail may pass mutliple versions:

    ad-trail-bundle
    - trail-id
    - version-id

## work in progress trails

We need to log when a trail is split (update) or cut off (delete, then `new-id-after-cut` is `null`)

    ad-trail-cuts
    - original-trail-id
    - new-id-before-cut
    - new-id-after-cut

We could either propagate such cuts to `ad-trail-bundle` within the transaction that creates them.  Or we could pull them into `ad-trail-bundle` before we look at some previous version' data.  Or we could follow a mixed strategy, polling this table until it becomes cumbersome, then do some housekeeping and püush it out to all versions and vacuum `ad-trail-cuts`.

Thus this table is a housekeeping, wip-tracking aid, not static data for eternity.

## what's needed in the data tables

The only field that is added to the data tables is the `ad-trail-id` 

    some-data-table-1
    - ...
    - ad-trail-id
    - ...
  
And another one:

    some-data-table-2
    - ...
    - ad-trail-id
    - ...

All queries narrow down to the trails of the requested version, like this:

    SELECT *
    FROM
      some-data-table-1, some-data-table-2, ..., 
      (select trail-id from ad-trail-bundle where version-id == 'REQUESTED_VERSION') AS trail-ids
    WHERE
      ...
      AND
      t1.ad-trail-id in trail-ids
      
# automating the process:  triggers

## ON INSERT
## ON DELETE
## ON UPDATE

## what can not be automated with triggers

  - `ad-trail-cuts` housekeeping
  - minimize the number of trails of frequently used queries

# future
* merge
* several sessions on the same version?
