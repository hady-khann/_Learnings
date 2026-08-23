# community.general.ceph_pool — Field Reference

For the authoritative, version-matched documentation on the node itself, always run:

```bash
ansible-doc community.general.ceph_pool
```

## Fields

### `name` (required)
Name of the pool. Example: `mypool`

### `state`
- `present` — creates the pool if it doesn't exist (default)
- `absent` — deletes the pool (**caution**: this destroys data)
- `list` — only lists existing pools, makes no changes

### `pool_type`
- `replicated` — the default type. Each object is stored as multiple full copies (replicas) across different OSDs. Suitable for most use cases (RBD, general CephFS).
- `erasure` — instead of full replicas, data is split into chunks plus parity. Uses less space but has higher processing overhead, so it's less recommended for latency-sensitive workloads (like interactive RBD) and more suited to archival/RGW use cases.

### `size`
Replicated pools only. Number of copies of each object (including the original). Example: `size: 3` means each piece of data is kept on 3 different OSDs. If omitted, the cluster's `osd_pool_default_size` is used.

### `min_size`
Replicated pools only. The minimum number of healthy copies the pool needs before it will stop serving I/O. Usually `size - 1` or lower. If omitted, Ceph calculates it automatically.

### `erasure_profile`
Erasure pools only. The name of a predefined erasure-code profile (or `default`). The profile determines how many data chunks (k) and parity chunks (m) are used.

### `rule_name`
The name of the CRUSH rule the pool should follow (e.g. for separating by disk type — HDD/SSD — or rack awareness). If omitted, the default `replicated_rule` is used.

### `pg_num`
Number of Placement Groups. Directly affects data distribution and performance. For a small practice cluster (3 nodes), values like 8, 16, or 32 are sufficient. Rough official Ceph formula:

```
pg_num = (number_of_OSDs * 100) / size, rounded to the nearest power of 2
```

### `pgp_num`
The number of PGs actually used for placement (distribution calculation). Should generally equal `pg_num`; if omitted, it defaults to the same value as `pg_num`.

### `pg_autoscale_mode`
- `on` — Ceph automatically adjusts `pg_num` based on data volume over time (default in newer versions, and more convenient for a practice environment since you don't have to calculate it yourself)
- `warn` — only warns that `pg_num` is suboptimal, but doesn't change it
- `off` — fully manual; the `pg_num` value you set stays fixed

### `target_size_ratio`
When the autoscaler is on, this tells Ceph roughly what proportion of total cluster capacity this pool is expected to consume (e.g. `0.2` = 20%). Helps the autoscaler pick a more sensible `pg_num` from the start.

### `application`
The application that will use this pool; equivalent to running `ceph osd pool application enable`. Common values:
- `rbd` — block storage / VM disk images
- `cephfs` — distributed filesystem (usually set automatically when creating a CephFS, not manually)
- `rgw` — object storage / S3-compatible gateway

### `cluster`
The Ceph cluster name, if not using the default `ceph`. Not needed for most environments, including this Vagrant-based cluster.
