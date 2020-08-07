---

title: 'CockroachDB Database Configuration'
weight: 50
menu:
  main:
    identifier: "user-guide-process-engine-database-crdb-configuration"
    parent: "user-guide-process-engine-database"

---

This section of the documentation describes how to use the Camunda Process Engine with the [CockroachDB
database](https://www.cockroachlabs.com/).

CockroachDB is a highly scalable SQL database that operates as a distributed system. As such, it has
different requirements and behavior compared to the other Camunda-supported databases. For this reason, 
we have adjusted the Process Engine behavior, and added some additional mechanisms in order to make sure 
that the Process Engine is able to operate correctly on CockroachDB. 

# Communication with CockroachDB

Currently, CockroachDB uses the PostgreSQL JDBC driver for communication between a Java application, like
the Process Engine, and the database. It is recommended to use a driver version compatible with PostgreSQL
9.5+, i.e. versions 42.X.X of the JDBC driver (see [the CockroachDB docs](https://www.cockroachlabs.com/docs/v20.1/install-client-drivers.html#java) 
for more details).  

# Custom CockroachDB transaction retry mechanism

Whenever the Process Engine detects a concurrency conflict between transactions, it reports an
`OptimisticLockingException` (for more details please see [this docs section]({{< ref "/user-guide/process-engine/transactions-in-processes.md#the-optimisticlockingexception" >}})), 
which is then handled internally, or reported to the user. Since CockroachDB implements its own 
optimistic locking mechanism, it requires that concurrent transactions be completely rolled back 
and retried. For this reason, the Process Engine implements a CockroachDB-specific Command retry 
mechanism.

The number of CockroachDB Command retries can be configured with the `commandRetries` Configuration
property (more details [here][descriptor-tags]). By default, 0 retries are allowed since we believe 
that the number of allowed retires needs to be set by the use-case.

When the number of retries is exceeded, or is set to 0, a `CrdbTransactionRetryException` will be
reported to the user. This exception is a child class of the `OptimisticLockingException` class, and
signifies that CockroachDB detected a concurrency conflict. The transaction has been rolled back, and
a retry of the failed API invocation is required.

It is important to note that the CockroachDB retry mechanism is applied to Process Engine Commands
where an `OptimisticLockingException` is handled internally in the regular case. The mechanism is also
applied to Commands that use a pessimistic lock, which is redundant and disabled when CockroachDB is used
(please see [the section bellow](#pessimistic-locks-replacement-behavior) for more details). You can 
find a list of the specific Commands below:

* `BootstrapEngineCommand` - used as part of the Process Engine initialization on startup. 
* `DeployCmd` - used to deploy the BPMN/CMMN/DMN models and associeated resources.
* `HistoryLevelSetupCommand` - used to detect and set the history level. 
* `AcquireJobsCmd` - used to obtain and lock Jobs for the `JobExecutor`.
* `FetchExternalTasksCmd` - used to obtain and lock `ExternalTasks`.
* `HistoryCleanupCmd` - used to configure and schedule a Job for cleanup of historical data.


# Differences in Process Engine Behavior

This section covers the differences Process Engine behavior when it is used with CockroachDB 
versus usage with other Camunda-supported databases. It provides descriptions and 
recommendations on how to configure and use the Process Engine with CockroachDB.

## Unsupported Exclusive Message correlation

Exclusive Message correlation (see {{< javadocref page="?org/camunda/bpm/engine/runtime/MessageCorrelationBuilder.html#correlateExclusively()" text="Javadocs" >}}) 
prevents multiple concurrent message correlations to be performed on a single Process execution by
using pessimistic locks. When the Process Engine is used with CockroachDB, this is not possible since any concurrency conflict 
will cause the transaction encompassing the message correlation to be retried, and any business 
logic repeated. For this reason, it is not possible to use Exclusive Message correlation in 
combination with CockroachDB.

If the Exclusive Message correlation functionality is required, it is recommended to set the
`commandRetries` Configuration property (more details [here][descriptor-tags]) to 0, so that any
concurrency conflicts (a `CrdbTransactionRetryException`) may bubble up to the user code, and be
handled according to the use-case. 

## Un-ignorable historic `OptimisticLockingException`

The Process Engine may generate large amounts of historical data, and provides the [history cleanup
feature]({{< ref "/user-guide/process-engine/history.md#history-cleanup" >}}) to ensure that "old"
data is removed. The History Cleanup Removal-Time-based Strategy allows for historical data associated
with still running Process Instances to be cleaned up. Since running Process Instances continue
generating historical data, removing the same data in parallel is viewed as an `OptimisticLockingException`.

In the regular case, this historic `OptimisticLockingException` is handled internally and ignored 
since it is expected (and can be controlled by the `skipHistoryOptimisticLockingExceptions` 
[configuration property][descriptor-tags]). However, due to the `SERIALIZABLE` transaction isolation, 
CockroachDB detects this concurrency conflict by itself and requires for a complete transaction 
rollback and retry. 

Consequently, when CockroachDB is used, the Process Engine will not attempt to ignore the reported 
`CrdbTransactionRetryException`, but will report it to the user, so that the associated business 
logic may be retried.  

## Pessimistic locks replacement "behavior"

As mentioned [above](#custom-cockroachdb-transaction-retry-mechanism), the CockroachDB-specific retry
mechanism in the Process Engine replaces the usage of pessimistic locks in the following commands:

* `BootstrapEngineCommand` 
* `DeployCmd`
* `HistoryLevelSetupCommand` 
* `HistoryCleanupCmd`

The role of the pessimistic locks in these Commands is not to prevent concurrent modification on the
locked data, but to serve as barriers (or flags) to ensure the serialized, single execution of a
given action (e.g. multiple deployments, multiple process engine startups, etc). Since CockroachDB
ensures a serialized execution through its implementation of the `SERIALIZABLE` transaction isolation
level, the pessimistic locks are redundant, or even counter-productive, as they introduce an additional
waiting time overhead for each concurrent transaction that attempts to acquire the pessimistic lock.

As a result of this replacement, multiple Process Engine startups will take longer to finish, as each
concurrent startup will fail and be retried instead of waiting to acquire a pessimistic lock. The number 
of retires is defined by the `commandRetries` configuration property described in the [CockroachDB retry 
mechanism section](#custom-cockroachdb-transaction-retry-mechanism). Once the retires are exausted, the
startup will have to be retried manually by the caller (user). The same scenario can be applied to the
other Commands listed above.

## Using External Transaction management with the Spring/Java EE integrations

{{< note title="When to consider the below recommendations" class="info" >}}
This section explains how the Process Engine can be coupled with CockroachDB using external 
transaction management. The descriptions below can be applied to both the Spring, and Java EE 
Transaction integrations, as they operate on similar concepts.

Note that the configuration described below is only required when transactions involving Commands listed
[here](#custom-cockroachdb-transaction-retry-mechanism) are managed by the Spring/Java EE frameworks. 
When the Process Engine is used inside a Spring/Java EE application, but left to manage its own 
transactions, the recommendations from the sections above remain valid.
{{< /note >}}

The [Spring]({{< ref "/user-guide/spring-framework-integration/transactions.md" >}})/[Java EE]({{< ref "/user-guide/cdi-java-ee-integration/jta-transaction-integration.md" >}}) 
Transaction integrations enable developers to manage transactions through the respective framework. 
This means that the Process Engine doesn't control when transactions are started, committed, 
or rolled back. 

In the regular case, this works well. However, when CockroachDB is used and a concurrency conflict is
detected, the Process Engine's [CockroachDB-specific Command retry mechanism](#custom-cockroachdb-transaction-retry-mechanism)
will kick in. This mechanism assumes a rolled-back transaction, which doesn't fit with how the Spring/Java 
EE Transaction integration works. In cases like these, a transaction might start earlier, continue 
after the Process Engine API invocation, and be rolled back only when all the involved operations 
are finished.

To ensure that the Process Engine can be combined with CockroachDB and the Spring/Java EE Transaction
integrations, the `commandRetries` configuration property (documented [here][descriptor-tags])
should be set to 0. This will effectively disable the CockroachDB-specific Command retry mechanism, and
allow the `CrdbTransactionRetryException` to bubble up to the Spring/Java EE layer, where the transaction
rollback and retry can be correctly implemented by the application developers.

# Differences between CockroachDB and other databases

This section covers the differences in behavior between CockroachDB and other Camunda-supported
databases that impact how the Process Engine operates. As opposed to the differences described in
the [previous section](#differences-in-process-engine-behavior), this section describes more general
CockroachDB characteristics, for which an alternative is not available, or more difficult to 
implement.

## Unsupported Engine-side Query timeout

The `jdbcStatementTimeout` configuration property (more details [here][descriptor-tags]) enables users to set the amount of time in seconds 
that the JDBC driver will wait for a response from the database. However, CockroachDB currently doesn't
support query cancellation ([CockroachDB issue](https://github.com/cockroachdb/cockroach/issues/41335)), 
so the Process Engine `jdbcStatementTimeout` property will not have any effect when used on CockroachDB.

## Server-side transaction timeout of 5 minutes

A common feature of databases is to introduce a server-side timeout on long-running transactions. 
However, in CockroachDB this timeout is tied to the access time interval (lease) that CockroachDB nodes 
have on the data included in the transaction. Since the lease time is hard-coded to 5 minutes, transactions
running for more than 5 minutes may experience a server-side timeot with the error `RETRY_COMMIT_DEADLINE_EXCEEDED`.

In cases like these, it is advised to shorten the execution time of the business logic associated with
the transaction since setting a custom server-side transaction timeout in CockroachDB is currently not
possible ([CockroachDB issue](https://github.com/cockroachdb/cockroach/issues/51042)).

## Blocking Transactions on Database READ

CockroachDB implements the `SERIALIZABLE` transaction isolation level, under which any single, successful
transaction brings the database to the next "stable" state. Parallel transactions are not allowed since
this would violate this rule ([CockroachDB docs](https://www.cockroachlabs.com/docs/v20.1/demo-serializable.html)).

As a consequence, when a transaction attempts to READ data which is currently modified by a concurrent
transaction, the former will block until the latter is finished (committed). In the context of the
Process Engine, certain operations may take longer than expected. For example, attempting to view (READ) 
the runtime state of a Process Instance in Cockpit, while the `JobExecutor` is executing a Job associated 
with that Process Instance, will block the Cockpit transaction until the Job is completed. 

## Concurrency conflicts on unrelated data from the same table

The previous section explained how CockroachDB implements the `SERIALIZABLE` transaction isolation level,
and how it impacts concurrent access to the same data. In addition, the currently supported CockroachDB
version (v20.1.3), may view concurrent modifications of unrelated data of the same table as a conflict
([CockroachDB issue](https://github.com/cockroachdb/cockroach/issues/51648)).

We have found that this happens in cases where a foreign key constraint is used to reference data from 
the same table. In the Process Engine, this pattern is used in the `ACT_RU_EXECUTION` table, to 
determine the Process Instance (aka parent execution) of each Task (execution) used in a given process.
When a Process Instance is deleted by one transaction, while a second transaction performs a modification
on another, unrelated Process Instance, a `CrdbTransactionRetryException` may be reported. As a result,
the modification of the latter will have to be retired.

[descriptor-tags]: {{< ref "/reference/deployment-descriptors/tags/process-engine.md#configuration-properties" >}}