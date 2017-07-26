---
id: revision
title: Revision
permalink: guides/couchbase-lite/native-api/revision/index.html
---

Couchbase Lite uses revisions to resolve conflicts detected during replication. One significant difference from other databases is document versioning. Couchbase Lite uses a technique called Multiversion Concurrency Control (MVCC) to manage conflicts between multiple writers. This is the same technique used by version-control systems like Git or Subversion, and by WebDAV. Document versioning is similar to the check-and-set mechanism (CAS) of Couchbase Server, except that in Couchbase Lite versioning is required rather than optional and the token is a UUID rather than an integer.

Every document has a special field called `_rev` that contains the revision ID. The revision ID is assigned automatically each time the document is saved. Every time a document is updated, it gets a different and unique revision ID.

When you save an update to an existing document, you must include its current revision ID. If the revision ID you provide isn’t the current one, the update is rejected. When this happens, it means some other endpoint snuck in and updated the document before you. You need to fetch the new version, reconcile any changes, incorporate the newer revision ID, and try again.

Keep in mind that Couchbase Lite is not a version control system and you must not use the versioning feature in your application. They’re there only to help with concurrency and resolving conflicts during replication.

## Resolving Conflicts

Revisions form a tree data structure since they can have multiple branches. In the case where there are multiple branches, one or more conflicts exist and should be resolved per the application requirements. Refer to the [Adding Synchronization](../../../../training/develop/adding-synchronization/index.html#resolve-conflicts) lesson to learn how to resolve conflicts in your application.

## Tombstones

The reason that tombstone revisions exist is so that deletes can be sync'd to other databases. If revisions were simply deleted with a naive approach, then there would be no easy way to sync up with other databases that contained the revision.

There is a special field in a revision's JSON called `_deleted` which determines whether the revision is a tombstone revision or not. A consequence of this fact is that tombstone revisions can hold arbitrary amounts of metadata, which can be useful for an application. If the full metadata of the document is preserved in the tombstone revision, then a document could easily be restored to it's last known good state after it's been deleted at some point.

For examples of deleting revisions via adding a tombstone revision, refer to the guide on Documents.

## Saved vs Unsaved Revision

Here are the main differences between Saved and Unsaved Revision objects:

- Unsaved revisions have not yet been persisted to the database.
- Saved revisions have already been persisted to the database.
- Unsaved revisions are useful for adding attachments.

Unsaved Revisions are mainly useful for manipulating attachments, since they provide the only means to do so via the API. See Attachments for examples of adding/removing attachments.

## Pruning

Pruning is the process that deletes the metadata and/or JSON bodies associated with old non-leaf revisions. Leaf revisions are not impacted. The process runs automatically every time a revision is added. The **maxRevTreeDepth** value defaults to 20, which means that the metadata and JSON bodies of the last 20 revisions are retained in Couchbase Lite as shown on the animation below.

<img src="https://cl.ly/321B1Y3T0K07/pruning-cbl.gif" class=portrait />

If there are conflicting revisions, the document may end up with **disconnected branches** after the pruning process. In the animation below, the document has a conflicting branch (revisions `3'` - `7'`). When the current revision (or longest branch) reaches the 23rd update, the conflicting branch is cut off. The revision tree is not in a corrupted state and the logic that chooses the winning revision still applies. But it may make it impossible to do certain merges to resolve conflicts and occupy disk space that could have been free-ed if the conflict was resolved early on.

<img src="https://cl.ly/0q342b0R251y/pruning-conflict.gif" class=portrait />

## Compaction

Compaction is defined as the process of purging the JSON bodies of non-leaf revisions. As shown on the diagram below, only properties with a leading underscore (`_` is the character to denote properties reserved for Couchbase) are kept to construct the revision tree.

<img src="https://cl.ly/1Q1F0i3f2i3n/compaction.gif" class=portrait />

Compaction can only be invoked manually via the [compact()](../../../../references/couchbase-lite/couchbase-lite/database/database/index.html#void-compact) method. The compaction process does not remove JSON bodies of leaf nodes. Hence, it is important to resolve conflicts in your application in order to re-claim disk space when the compaction process is executed.