# zkLink Nova Node Docs

Operator documentation for running a zkLink Nova self-hosted RPC node (external node).

* [Run a Node](developer/run-a-node.md) — bootstrap a node from the mainnet database snapshot.

## Snapshot

The current mainnet snapshot, its checksum and its manifest are published at:

```
https://zklink-nova-en-snapshots.s3.ap-northeast-1.amazonaws.com/zklink_nova_en_snapshot.backup
https://zklink-nova-en-snapshots.s3.ap-northeast-1.amazonaws.com/zklink_nova_en_snapshot.backup.sha256
https://zklink-nova-en-snapshots.s3.ap-northeast-1.amazonaws.com/MANIFEST.txt
```

When a new snapshot is published, four things in `developer/run-a-node.md` need updating together:
the download URL, the SHA256, the content point (L2 block / L1 batch / timestamp), and the measured
rebuild timings.
