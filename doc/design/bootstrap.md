# Bootstrap Design

etcd currently uses SRV records to bootstrap while this is effective managing an external source of truth is not optimal. This design allows for any source of truth to be used containing a membership list.

## General design

Bootstrap assumes that no existing cluster exists. The bootstrap mechanism can only be used during the bootstrap phase of cluster lifecycle. The design has 2 phases `initialization` and `population`. Each of these phases is intended to be run on each node, thus bootstrapping each member separately.

## Goals

- Configure an etcd cluster by initializing the etcd v3 store and populating the member bucket.
- Create and deploy the etcd data-dir.

## Non Goals

- Reconfigure running cluster.
- Restore existing cluster, while we use clientv3/snapshot, bootstrap` will only be used to init a new cluster.
- Populating etcd server configurations.

## Initialization

Initialization generates a new v3 backend and populates only the member and cluster buckets.

```golang
// InitDB generates a blank etcd db file and returns the file path.
func InitDB(...) .. {
	if err := fileutil.TouchDirAll(c.Path); err != nil {
		return "", fmt.Errorf("create db directory error: %v", err)
	}
	dbFilePath := fmt.Sprintf("%s/%s", c.Path, c.Name)
	be := backend.NewDefaultBackend(dbFilePath)
	defer be.Close()
	mustCreateBackendBuckets(be)

	return dbFilePath, nil
}

func mustCreateBackendBuckets(be backend.Backend) {
	tx := be.BatchTx()
	tx.Lock()
	defer tx.Unlock()
	tx.UnsafeCreateBucket(membersBucketName)
	tx.UnsafeCreateBucket(clusterBucketName)
}
```

## Population

Population uses the familiar snapshot restore process to populate the member bucket of the datastore.
This allows us to populate the member list with any discovery mechanism we choose. The other advantage is that
support for this code is already included in etcd golang client clientv3.

```golang
    // Populate takes blank etcd db and populates member list and deploys to data-dir.
    func Populate(...) .. {
        restorePeerURLs := "http://test1.local:2380"
        restoreCluster := "test1=test1.local:2380,test2=test2.local:2380,test3=test3.local:2380"
        initDBPath := "path/to/etcdInitDB"
        dataDir := "path/to/dataDir"
        walDir := filepath.Join(dataDir, "member", "wal")
        restoreClusterToken := clusterID

        sp := snapshot.NewV3(lg)

        if err := sp.Restore(snapshot.RestoreConfig{
	    SnapshotPath:        initDBPath,
	    Name:                restoreName,
	    OutputDataDir:       dataDir,
	    OutputWALDir:        walDir,
	    PeerURLs:            strings.Split(restorePeerURLs, ","),
	    InitialCluster:      restoreCluster,
	    InitialClusterToken: restoreClusterToken,
	    SkipHashCheck:       true,
        }); err != nil {
	// handle error
        }
    }
```
