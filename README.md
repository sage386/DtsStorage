# DtsStorage 
Is an in-memory tree-like lightning fast storage container with the features described below.
Copyright (C) 2017 Alex A. Petroukine, sage386@hotmail.com

# Sample Usage
```sh
CStorage storage;

[thread 1]
{
    CTransaction tx(storage);                           // acquires a read-snapshot over current storage, zero time!
    int32_t value = tx["path"]["value"].getInt32();     // snapshot remains consistent and isolated from other thread modifications
    auto variant = tx["path"];                          // shortcuts to nodes
    for (auto i: variant) {...}                         // iterates over keys within a non-leaf node
}

[thread 2]
{
    CTransaction tx(storage, CTransaction::Default);    // acquires a read-snapshot as in prior example with no changes visibility
    tx["path"]["subpath"]["leaf"] = "string value";     // request to write/modify data within a storage.
    if (tx["value"] < 10)                               // with Default isolation mode this reads data only from snapshot
        tx["value"] = 10;                               // puts a record into transaction log to change the value
                                                        // if we it read again, we would read unchanged value with Default isolation mode
    // tx.commit();                                     // when tx goes out of scope or commit is manually called, changes applied atomically
}
```

# Features

1. Transactions

Changes to the tree are made within transactions.
All changes are either applied or rolled back atomically. Storage readers will never see intermediate changes.

2. Zero time snapshot

Read transactions read data from a "snapshot" taken over the whole storage during the time of read transaction creation.
The snapshot is _very_ fast operation that actually consist of adding a single reference to the current storage version.
Transaction snapshot is fully consistent and is completely isolated from changes made by other transactions.

3. Isolation

For a transaction it is possible to specify read-isolation level.
Default level allows reading snapshot data only, ReadUncommited level allows reading this transaction uncomitted-yet-data
merged with snapshot (as if commited). ReadUncomittedOnly reads uncomitted changes only for the current transaction.

4. Versioning

Every change in a tree is an allocation of a new object.
Every new object is stamped with a generation id, which is an uint64 increasing counter.
If there is an update to existing value, previous object is auto-disposed when there are no more snapshots that may read it.

5. Block-free

Both read and write transactions are block-free!!!
Although Read transactions are almost fast as lock-free code, there are actually read-write capable spinlocks for rehash purposes only.
By block-free I merely mean that the API provides no way for one transaction to block indefinitely any other.

6. Atomicity

Any transaction changes to the tree are actually put into transaction log first (which has the same structure as the main tree).
So, threads may write data (actually only prepare their changes) in parallel, not blocking each other.
Only during the commit there is a lock guarding applying of transaction log to the main storage.
Since everything is allocated and prepared outside the commit, the commit operation comprises only of proper re-linking of structures.
Disposals are also managed after the commit lock.

When there are concurent write transactions to the same place in the storage, the first transaction that issues a commit will 
apply it's changes first. Thus writers to the same place of the storatge should synchronize with each other themselves.

7. Iterators

For non-leaf node, it's keys can be enumerated using forward only iterator.

8. Lock-Free API

Users of a storage gain clean, lock-free API.
It's never been easier acessing data for both read and write in concurent manner.

9. Change Notifications

Subscription on node changes and dispatching change notifications is built in.
Notifications work asynchronously outside of commit and only provide indication of changes occurance.
The actual changes are not reported though. Change notification handler is passed a version reference that is guaranteed to have changes.

10. Replication Layer

An Endpoint layer abstracted from transport is also there. It allows an endpoint to make shares of specific paths of a storage(s),
thus exposing/publish them for replication. Recursive sharing for subtree structure is also supported. 
Other endpoints easily can subscribe for those shared resources and effectivelly receive autimatically replicated data into its local DstStorage folder.

11. Lightning Fast

Designed for speed but kept also space efficiency in mind.

...more


