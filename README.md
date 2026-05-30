# shard-manifest-helm

Helm chart for [`shard-manifest`](https://github.com/lightwebinc/shard-manifest)
— the BRC-137 Shard Manifest Announcement Daemon.

The daemon periodically multicasts a small datagram to the IPv6 beacon
group (`FF0X::B:FFFD`) advertising the local participant's `shard_bits`
configuration and the set of shard groups it has joined. It is purely
informational and does not subscribe to or interpret data-plane traffic.

## Quick start

```bash
helm install my-manifest oci://ghcr.io/lightwebinc/charts/shard-manifest \
  --version 0.1.0 \
  --namespace bsv-mcast --create-namespace \
  --set manifest.shardBits=4 \
  --set manifest.joinedGroups=all \
  --set manifest.roleHint=proxy
```

## Networking

The chart supports three networking modes:

| `networking.mode` | Use when                                                        |
| ----------------- | --------------------------------------------------------------- |
| `pod` (default)   | Cluster CNI provides IPv6 multicast egress to the beacon group. |
| `host`            | Pin to nodes with the fabric NIC; uses host network namespace.  |
| `multus`          | Attach a NetworkAttachmentDefinition for the fabric NIC.        |

When using `multus`, set `manifest.iface` explicitly to the secondary
interface name (default `net1`).

## Probes

| Endpoint   | Probe       | Default port |
| ---------- | ----------- | ------------ |
| `/healthz` | liveness    | 9091         |
| `/readyz`  | readiness   | 9091         |
| `/metrics` | scrape only | 9091         |

`/readyz` returns 200 once a manifest has been sent within
`2 × manifest.announceInterval` and 503 otherwise (starting / draining /
stale).

## Prometheus

Set `metrics.serviceMonitor.enabled=true` to render a `ServiceMonitor`
(requires the prometheus-operator CRD).

## Values

See [`values.yaml`](values.yaml) for all options, validated against
[`values.schema.json`](values.schema.json).

### SSM (Source-Specific Multicast)

`manifest.sourceMode` defaults to `asm`. When `ssm`:

- `manifest.publishers` MUST be a non-empty list of IPv6 literals
  and/or DNS names — typically a headless-Service name fronting the
  shard-proxy pods. Resolved via `shard-common/bootstrap.Resolver`
  and emitted as the `Flags.SourcesValid` payload union (BRC-137
  bit 4). `manifest.publishersRefresh` (default `30s`) sets the DNS
  re-resolve interval; last-good AAAAs are retained on transient
  failures.
- Every emitted manifest sets `Flags.SourceModeSSM` (BRC-137 bit 3)
  so listeners switch their data-plane address derivation to the
  `FF3x::/32` SSM prefix.
- The shard-manifest pod's own per-pod IPv6 (Multus + Whereabouts)
  is what receivers list in their `sources.bootstrap.manifest` to
  `(S,G)`-join the manifest group under Posture C.

See the [SSM Support Plan](https://github.com/lightwebinc/bsv-multicast/blob/main/docs/SourceSpecificMulticast/ssm-support-plan.md)
for fabric prerequisites.

## License

Apache 2.0. See [LICENSE](LICENSE) and [NOTICE](NOTICE).
