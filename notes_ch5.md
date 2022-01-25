# Chapter 5: Replication

reasons you want to have replications:

- keep data close to customer (lower latency)
- allow system to continue to work when failover (increase availability)
- scale out number of machines that can serve read queries (increase read throughput)

## Leaders and Followers

1. One of replicas is designated the leader (or master/primary). When clients want to write to databse, they must send their requests to the leader, which firs writes the new data to its local storage
2. followers (read replicas) takes log from the leader and updates its local copy of the databse.
3. When a client wants to read from the database, it can query either the leader or any of followers. But writes area only accepts on the leader.

This usage is on

- RMDB: MYSQL, PostgreSQL
- NOSQL: MongoDB, RethinkDB and Espresso

## Sychronous vs Asynchronous Replication

### Setting up new followers

1. Take a consistent snapshot of the leder's databse at some point in time
2. copy the snapshot to the new follower node
3. The follower connects to the leader and requests all the data changes that have happened since the snapshot was taken. This requres that the snapshot is associated with an exact position in the leader's replication log.
4. When the follower has processed the backlog of data changes since the snapshot, we say it has caught up
