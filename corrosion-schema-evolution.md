I think schema evolution here may be better treated primarily as a **deployment and compatibility problem**, rather than as a distributed-consensus problem inside Corrosion itself.

Corrosion is deliberately optimized around local availability and eventual convergence. Introducing a consensus protocol into the normal replication path just to coordinate schema changes would add a substantially different consistency model to the system.

A simpler approach might be to define two things explicitly:

## 1. Schema deployment protocol

A deployment hook such as [`release_command`](https://fly.io/docs/reference/configuration/#the-deploy-section) could be used as an orchestration entry point, but it should not itself be considered the schema distribution mechanism, since it runs only once in a temporary Machine.

Conceptually, the rollout could look like this:

```text
[ Deployment / Control Plane ]
            |
            v
1. Publish schema version N+1
            |
            v
2. Distribute and apply N+1 to Corrosion nodes
            |
            v
3. Deploy application version N+1
            |
            v
4. Enforce schema compatibility locally as Corrosion nodes
   and application workers become ready for N+1 traffic
            |
            v
5. Local writers start using N+1 features only after
   local compatibility has been established
```

The important invariant is:

> Global agreement on schema readiness is not required; each participant must establish local compatibility before becoming active for traffic that depends on that schema.

This keeps schema coordination separate from the normal data-replication path and avoids requiring schema propagation itself to have the same semantics as application-data gossip.

The exact distribution mechanism could remain an implementation detail: control-plane configuration, a dedicated schema record/service, configuration distribution, etc.

## 2. Compatibility contract

Schema changes could follow an **expand / contract** lifecycle.

During the **expand phase**, migrations should remain backward-compatible where possible:

- add a nullable column;
- add a new table;
- introduce a new representation alongside the old one;
- allow old and new application versions to coexist during a rolling deployment.

Only after old readers and writers have been drained should the system enter an explicit **contract phase**:

- drop an old column;
- remove an obsolete table;
- remove or rename an old representation;
- perform other compatibility-breaking changes.

This seems consistent with the direction discussed in [#167](https://github.com/superfly/corrosion/issues/167), where destructive schema changes require explicit authorization rather than being performed as part of a normal reload.

## Offline / stale nodes

Waiting for every node in the cluster before completing a schema deployment would make progress depend on unavailable nodes, which seems undesirable for an AP/eventually-consistent system.

Instead, a node returning after being offline could establish schema compatibility **before becoming an active replication participant**.

For example:

```text
node rejoins
     |
     v
discover current schema requirements
     |
     +-- compatible -----------------> join normally
     |
     +-- stale / incompatible
              |
              v
       obtain and apply required schema
              |
              +-- success -----------> join
              |
              +-- cannot upgrade ----> quarantine / re-bootstrap
```

The current schema requirement could be learned during admission, through control-plane state, membership metadata, a dedicated schema record, or another mechanism.

The important part is that a stale node should ideally detect incompatibility **before** discovering it by failing to apply replicated business data.

This suggests a slightly weaker but more useful invariant than:

> Every node must always have exactly the same schema.

Namely:

> A node must not generate or apply replicated data unless its local schema is compatible with the requirements of that data.

That also leaves room for mixed-schema operation during safe expand phases while still preventing incompatible writers/readers from entering the **active replication set**.

## Open questions

A few things seem worth defining explicitly if this model is useful:

- What is the source of truth for the current schema version?
- What exactly constitutes “compatible” — schema version, capabilities, or something more granular?
- Which schema changes are safe during mixed-version operation?
- At what point is an old schema version no longer allowed to rejoin?
- Should incompatible changes require re-bootstrap rather than in-place migration?
- Does Corrosion already have an internal deployment assumption that makes mixed-schema operation unnecessary?

Would this roughly match how schema changes are handled internally at Fly, or is there an existing invariant in Corrosion that would make this model unnecessary?