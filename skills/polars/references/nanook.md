# Polars Nanook: Remote and Distributed Queries

Polars Nanook runs the same lazy Polars query on one large machine or
distributed over a cluster: `.collect()` becomes `.remote(ctx).collect()`,
with no other changes. It also profiles queries that run locally with
open-source Polars, with no cluster involved. Nanook replaces the
products formerly called Polars Cloud (now the AWS deployment) and Polars
On-Prem (now the Helm chart, bare-metal and Ray deployments).

Use it when the user mentions Nanook, Polars Cloud, Polars On-Prem,
`polars_nanook`, `polars_cloud`, a `ComputeContext` or `ClusterContext`,
when a query outgrows one machine (`references/memory.md`), or when they
want to know why a query is slow or where its time and memory go. Everything in
SKILL.md still applies: build one lazy query, then execute it once.

## Contents

Install and authenticate; profiling local queries; contexts; running a
remote query; results; distributed vs single node; data, packages and
versions; memory; old names; docs map.

## Install and authenticate

```bash
pip install polars polars-nanook      # CLI: `nanook`
```

`polars-nanook` pins the exact `polars` version it was built for, so do not
upgrade `polars` separately in that environment.

```python
import polars as pl
import polars_nanook as pn

pn.authenticate()                     # cached token, else browser login
pn.authenticate(interactive=False)    # CI: fail instead of opening a browser
```

Headless jobs use a service account: set `NANOOK_CLIENT_ID` and
`NANOOK_CLIENT_SECRET`, or `NANOOK_ACCESS_TOKEN`. A `ClusterContext` needs
no login.

## Profiling local queries

`pl.Config.enable_monitoring()` is the query profiler for open-source
Polars. The query still runs on the user's machine; Polars sends its plan
and per-operator metrics to a Nanook workspace, where they show up in the
*Queries* tab. Reach for it when a query is slow or uses too much memory.
`LazyFrame.profile()` was removed in Polars 2.0, so do not suggest it.

```python
import polars as pl

pl.Config.enable_monitoring()                             # every query from now on
pl.Config.enable_monitoring(workspace="My workspace")     # add organization= if the name is ambiguous

with pl.Config(enable_monitoring=True):                   # only inside the block
    lf.collect()

lf.collect(engine=pl.StreamingEngine(monitoring=True))    # one query
```

Or turn it on without changing the code:

```bash
POLARS_QUERY_MONITORING=1 POLARS_QUERY_MONITORING_WORKSPACE="My workspace" python job.py
```

What the profile shows for each query:

- a timeline with planning time and run time;
- the optimized logical plan;
- the physical plan, with rows, morsels, CPU time and I/O time per
  operator, updated while the query runs;
- flags on operators: "Memory intensive", and "In-memory fallback" for
  operations the streaming engine cannot run (`references/memory.md`).

Requirements and side effects:

- A Nanook account, and `polars-nanook` installed in the same environment.
  Without the package, enabling monitoring raises an error asking to
  install it.
- `polars-nanook` pins one exact `polars` version, so installing it can
  change the installed Polars. Monitoring works only on that pairing.
- The first monitored query logs in through the browser. Log in first
  with `nanook login`, or with `pn.authenticate(interactive=False)` plus
  service-account variables in CI (see above).
- Monitoring switches queries to the streaming engine, because only that
  engine collects per-operator metrics. Turning it off does not switch
  back; a `with pl.Config(enable_monitoring=True):` block does.
- Everyone in the workspace can see the profiles. The plan includes column
  names, file paths and literal values from filters, so check with the
  user before monitoring sensitive queries.

## Contexts

A context says where remote queries run.

```python
# AWS: Nanook starts EC2 instances in the workspace's AWS account
ctx = pn.ComputeContext(workspace="My workspace", cpus=8, memory=32, cluster_size=4)
ctx = pn.ComputeContext(workspace="My workspace", instance_type="m7i.xlarge")

# a cluster you run (Helm chart, bare-metal): connect to its scheduler
ctx = pn.ClusterContext(uri="https://scheduler.internal")
# Ray: RayComputeContext from the polars_nanook_ray package
```

- `ComputeContext` arguments are keyword-only. `memory` and `storage` are
  GB, and `cpus`/`memory` are lower bounds. `cluster_size` defaults to 1.
- With no size given, a `ComputeContext` uses the workspace defaults.
- An AWS cluster starts on the first query (about a minute) and keeps
  running until `idle_timeout_mins` without queries. Unset, it uses the
  workspace default (60 unless an admin changed it); the minimum is 10.
- Stop it when done:
  - `with pn.ComputeContext(...) as ctx:` makes `ctx` the default inside
    the block and stops the cluster on exit.
  - `ctx.stop()` stops it explicitly.
  - `ctx.start(wait=True)` starts it ahead of the first query.
- `pn.set_compute_context(ctx)` sets a default, after which
  `lf.remote().collect()` needs no argument. It also works as a `with`
  block or a decorator.

## Running a remote query

```python
lineitem = pl.scan_parquet("s3://bucket/lineitem/")    # storage the workers can read

query = (
    lineitem.filter(pl.col("l_shipdate") > pl.date(1995, 3, 15))
    .group_by("l_returnflag")
    .agg(pl.col("l_extendedprice").sum())
)

df = query.remote(ctx).collect()                       # small result -> DataFrame

query.remote(ctx).sink_parquet("s3://bucket/out/").await_result()   # large result
```

`pl.sql(...)` and `SQLContext.execute(...)` return LazyFrames, so
`.remote(ctx)` works on SQL the same way.

`lf.remote(ctx, ...)` takes:

- `engine`: `"auto"`, `"streaming"`, `"in-memory"` or `"gpu"`.
- `n_retries`: retries for failed tasks.
- `priority`.

## Results

- `.collect()` gathers the whole result on one worker and sends it to the
  client. Use it only for results that fit in memory.
- Sink large results: `.sink_parquet`, `.sink_csv`, `.sink_ipc` and
  `.sink_iceberg` write from the workers.
  - A sink returns as soon as the query is submitted;
    `.await_result()` waits and raises if the query failed.
  - A path without an extension (`s3://bucket/out/`) is a directory where
    every worker writes its own files.
  - A single file (`out.parquet`) needs `sink_to_single_file=True` and is
    written by one worker. Without the flag, the sink raises `ValueError`.
- `.show()`, `.execute()` and `.await_and_scan()` keep the result in the
  cluster's results location. On AWS it is set up automatically and kept
  for one day. On a cluster you run, they need an "anonymous results"
  location configured.
- `.remote(ctx).explain()` and `.show_graph()` show the distributed plan
  without running it.

## Distributed vs single node

Remote queries run on the distributed engine by default: stages connected
by shuffles, spread over the workers.

```python
query.remote(ctx).single_node().collect()                       # one worker
query.remote(ctx).distributed(min_workers=2, max_workers=8).sink_parquet(...)
```

- `NANOOK_SINGLE_NODE=1` or `pn.Config.set_single_node()` forces single
  node globally.
- `.distributed()` tunes the run with `partitions_per_worker`,
  `equi_join_broadcast_limit` and
  `single_worker_ops="auto"|"allow"|"forbid"`.

Some parts of a distributed query run on a single worker:

- `map_batches` without `is_elementwise=True`
- data created in the client (`pl.LazyFrame({...})`, `df.lazy()`)
- a single CSV or NDJSON file. A large Parquet file is split across workers
  by row group.

Prefer native expressions and Parquet. The query profile marks
single-node stages, and marks operations that fell back to the in-memory
engine.

## Data, packages and versions

- **Scans and sinks run on the workers.** Paths must be object storage
  (`s3://`, `gs://`, `az://`, ...) or a filesystem shared by every worker.
  A local path fails with `InvalidOperationError: 'file://' uri not
  supported as scan`: upload the data to object storage first.
  - On AWS, the workers read S3 through their IAM role.
  - For other storage, pass `storage_options` or a `credential_provider`.
- **Python UDFs and plugins** need their packages on the workers as well as
  the client. On AWS, pass `requirements="requirements.txt"` to
  `ComputeContext`; on a cluster you run, install them in the worker image.
- **Versions must match.** For a `ClusterContext`, the client's `polars`
  must match the cluster's release, and Python must match to the patch
  version because UDFs are pickled. Mismatches show up as
  "deserialization failed" errors.

## Memory

Each worker spills to disk like local Polars (`references/memory.md`).

- **AWS:** size machines with `memory=` or `instance_type=`. For heavy
  spills and shuffles, pick an instance type with local NVMe storage (for
  example `m8id.*`); otherwise spills go to a slow gp3 volume.
- **Helm chart:** `resources.limits.memory`.
- **Bare-metal:** `memory_limit` in bytes, as a plain integer. It needs a
  delegated cgroup, otherwise you get "failed to enforce memory limits".
- **Ray:** `worker_overrides={"memory_limit": BYTES}`.

Spilling starts at 80% of the limit. A task near the limit is aborted, and
the failed stage is rescheduled. On a query that runs out of memory, try
these in order:

1. `.distributed(partitions_per_worker=4)`
2. More workers
3. Bigger workers, sized from the per-stage worker memory in the profile of
   a run on a smaller sample.

## Old names

Old code may use these names. They still work for now; write the new ones.

| Old | New |
|---|---|
| `pip install polars-cloud` | `pip install polars-nanook` |
| `import polars_cloud as pc` | `import polars_nanook as pn` |
| `pc` CLI | `nanook` CLI |
| `POLARS_CLOUD_CLIENT_ID` etc. | `NANOOK_CLIENT_ID` etc. |
| `polars_cloud_ray.RayClusterContext(config)` | `polars_nanook_ray.RayComputeContext(...)` (no compatibility layer) |

Polars' own storage variables, such as `POLARS_CLOUD_MAX_RETRIES`, keep
their names.

## Docs map

Fetch `https://docs.nanook.pola.rs/<path>/` when you need more than this
file:

| Need | Path |
|---|---|
| Quickstart, local profiling | `get-started` |
| Reading a query profile | `run/query-profile` |
| Choose AWS / Helm / bare-metal / Ray | `get-started/choose-a-deployment` |
| Login, service accounts | `run/authenticate` |
| `ComputeContext` / `ClusterContext` | `run/compute-context`, `run/cluster-context` |
| `.remote()`, sinks, stored results | `run/remote-query`, `run/stored-results` |
| Cloud storage credentials | `run/storage` |
| Python packages on workers | `run/packages` |
| Distributed engine, scaling, workers | `run/distributed-engine`, `run/scale-out` |
| Worker memory limits | `deploy/configure/workers` |
| Error messages | `reference/errors` |
| Cluster releases and pinned Polars versions | `releases` |
