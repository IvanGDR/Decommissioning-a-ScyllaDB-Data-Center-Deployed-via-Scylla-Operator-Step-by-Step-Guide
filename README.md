# Decommissioning-a-ScyllaDB-Data-Center-Deployed-via-Scylla-Operator-Step-by-Step-Guide

Following ScyllaDB Operators series:
https://github.com/IvanGDR/Install-Scylla-Operator-in-on-premise-K8s-Cluster-and-deploy-multiDC-cluster
It may be required to decommissioned a DC.


### Decommission steps

1. put all nodes in the Decommissioned DC under maintenance mode
2. perform the repairs as per the guide
3. alter keyspaces to disable replication to the Decommissioned DC
4. while there are racks: scale a rack down to 0
5. edit all other scyllaclusters to remove the external seed
6. delete the SC


This is the initial cluster Set up:

```
$ kubectl --context="${CONTEXT_DC2}" -n=scylla2 exec -it pod/scylla-cluster-europe-west2-c-dc2-c-0 -c=scylla -- nodetool status

Datacenter: europe-west2-b-dc1
==============================
Status=Up/Down/eXcluded
|/ State=Normal/Leaving/Joining/Moving
-- Address        Load    Tokens Owns Host ID                              Rack
UN 192.168.49.209 1.25 MB 256    ?    0aea3d21-13b4-452e-a177-606b1c752f1b b   
Datacenter: europe-west2-c-dc2
==============================
Status=Up/Down/eXcluded
|/ State=Normal/Leaving/Joining/Moving
-- Address         Load      Tokens Owns Host ID                              Rack
UN 192.168.147.149 940.95 KB 256    ?    f381543d-c737-4ce6-8aa4-94d6eae6ee3c c   
```

And these are the scyllaclusters objects deployed as `scylla` DC and `scylla2` DC


**Full dc1.yaml file**

```
apiVersion: scylla.scylladb.com/v1
kind: ScyllaCluster
metadata:
  name: scylla-cluster
  namespace: scylla
spec:
  agentVersion: 3.11.2
  version: 2026.2.2
  cpuset: true
  automaticOrphanedNodeCleanup: true
  exposeOptions:
    broadcastOptions:
      clients:
        type: PodIP
      nodes:
        type: PodIP
    nodeService:
      type: Headless
  datacenter:
    name: europe-west2-b-dc1
    racks:
    - name: b
      members: 1
      storage:
        storageClassName: scylladb-local-xfs
        capacity: 100G
      agentResources:
        requests:
          cpu: 100m
          memory: 250M
        limits:
          cpu: 100m
          memory: 250M
      resources:
        requests:
          cpu: 2
          memory: 5G
        limits:
          cpu: 2
          memory: 5G
      placement:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - topologyKey: kubernetes.io/hostname
            labelSelector:
              matchLabels:
                app.kubernetes.io/name: scylla
                scylla/cluster: scylla-cluster
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values:
                - europe-west2-b
              - key: scylla.scylladb.com/node-type
                operator: In
                values:
                - scylla
        tolerations:
        - effect: NoSchedule
          key: scylla-operator.scylladb.com/dedicated
          operator: Equal
          value: scyllaclusters
```

**Full dc2.yaml file**

```
apiVersion: scylla.scylladb.com/v1
kind: ScyllaCluster
metadata:
  name: scylla-cluster
  namespace: scylla2
spec:
  agentVersion: 3.11.2
  version: 2026.2.2
  cpuset: true
  automaticOrphanedNodeCleanup: true
  exposeOptions:
    broadcastOptions:
      clients:
        type: PodIP
      nodes:
        type: PodIP
    nodeService:
      type: Headless
  externalSeeds:
  - 192.168.49.198 
  datacenter:
    name: europe-west2-c-dc2
    racks:
    - name: c
      members: 1
      storage:
        storageClassName: scylladb-local-xfs
        capacity: 100G
      agentResources:
        requests:
          cpu: 100m
          memory: 250M
        limits:
          cpu: 100m
          memory: 250M
      resources:
        requests:
          cpu: 2
          memory: 5G
        limits:
          cpu: 2
          memory: 5G
      placement:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - topologyKey: kubernetes.io/hostname
            labelSelector:
              matchLabels:
                app.kubernetes.io/name: scylla
                scylla/cluster: scylla-cluster
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values:
                - europe-west2-c
              - key: scylla.scylladb.com/node-type
                operator: In
                values:
                - scylla
        tolerations:
        - effect: NoSchedule
          key: scylla-operator.scylladb.com/dedicated
          operator: Equal
          value: scyllaclusters
```



#### steps

**1.- Stop client connecting to any node in the DC to be decommissioned**
The best is to put all node(s) to be decommissioned in maintenance mode
https://operator.docs.scylladb.com/stable/operate/use-maintenance-mode.html

**a)** Find service corresponding to your ScyllaDB node
```
$ kubectl -n scylla get svc

scylla-cluster-europe-west2-b-dc1-b-0
```

and

```
$ kubectl -n scylla2 get svc      

scylla-cluster-europe-west2-c-dc2-c-0
```

This is a node `scylla-cluster-europe-west2-c-dc2-c-0` from the DC planning to be decommisioned

**b)** Enable maintenance mode on each node to be decommissioned
```
$ kubectl -n scylla2 label svc scylla-cluster-europe-west2-c-dc2-c-0 scylla/node-maintenance=""
```
*** to disable it
```
$ kubectl -n scylla2 label svc scylla-cluster-europe-west2-c-dc2-c-0 scylla/node-maintenance-
```

**2) run the nodetool repair on each node in the data-center that is going to be decommissioned, one at a time in a rolling fashion**

```
$ kubectl --context="${CONTEXT_DC2}" -n=scylla2 exec -it pod/scylla-cluster-europe-west2-c-dc2-c-0 -c=scylla -- nodetool repair
```

**3) alter ks to disable replication on dc to be decommissioned**
get into a node:
```
$ kubectl -n scylla2  exec --stdin --tty scylla-cluster-europe-west2-c-dc2-c-0 -- /bin/bash
```
then via cqlsh execute to identify the Replication on all ks
```
cqlsh> select * from system_schema.keyspaces;
```

for those keyspaces (system and user ones) identify the ks schema
```
cqlsh> describe keyspace <keyspace name>;
```
example:
```
CREATE KEYSPACE ks_test WITH replication = {'class': 'org.apache.cassandra.locator.NetworkTopologyStrategy', 'europe-west2-b-dc1': '1', 'europe-west2-c-dc2': '1'} AND durable_writes = true AND tablets = {'enabled': false};
```
then alter the keyspace(s) accordingly
```
ALTER KEYSPACE ks_test WITH replication = {'class': 'org.apache.cassandra.locator.NetworkTopologyStrategy', 'europe-west2-b-dc1': '1'} AND durable_writes = true AND tablets = {'enabled': false};
```
*** Do it for all keyspaces (system and user ones)

**4) while there are racks: scale a rack down to 0**

https://operator.docs.scylladb.com/stable/operate/scale-add-remove-racks.html

Edit in-place (recommended for most changes):

```
$ kubectl -n scylla2 edit scyllaclusters.scylla.scylladb.com scylla-cluster
```
From:
```
apiVersion: scylla.scylladb.com/v1
kind: ScyllaCluster
metadata:
  creationTimestamp: "2026-08-10T16:47:12Z"
  generation: 1
  name: scylla-cluster
  namespace: scylla2
  resourceVersion: "119456"
  uid: c2c9edb6-598c-409f-98e7-2eadb7b65e14
spec:
  agentRepository: docker.io/scylladb/scylla-manager-agent
  agentVersion: 3.11.2
  automaticOrphanedNodeCleanup: true
  cpuset: true
  datacenter:
    name: europe-west2-c-dc2
    racks:
    - agentResources:
        limits:
          cpu: 100m
          memory: 250M
        requests:
          cpu: 100m
          memory: 250M
      members: 1          <---- will be changed to 0
      name: c

```
to

```
apiVersion: scylla.scylladb.com/v1
kind: ScyllaCluster
metadata:
  creationTimestamp: "2026-08-10T16:47:12Z"
  generation: 1
  name: scylla-cluster
  namespace: scylla2
  resourceVersion: "119456"
  uid: c2c9edb6-598c-409f-98e7-2eadb7b65e14
spec:
  agentRepository: docker.io/scylladb/scylla-manager-agent
  agentVersion: 3.11.2
  automaticOrphanedNodeCleanup: true
  cpuset: true
  datacenter:
    name: europe-west2-c-dc2
    racks:
    - agentResources:
        limits:
          cpu: 100m
          memory: 250M
        requests:
          cpu: 100m
          memory: 250M
      members: 0
      name: c
```

save and then:
```
$ kubectl -n scylla2 edit scyllaclusters.scylla.scylladb.com scylla-cluster

scyllacluster.scylla.scylladb.com/scylla-cluster edited
```

furthermore:

```
$ kubectl --context="${CONTEXT_DC1}" -n=scylla exec -it pod/scylla-cluster-europe-west2-b-dc1-b-0 -c=scylla -- nodetool status


Datacenter: europe-west2-b-dc1
==============================
Status=Up/Down/eXcluded
|/ State=Normal/Leaving/Joining/Moving
-- Address        Load      Tokens Owns Host ID                              Rack
UN 192.168.49.209 766.26 KB 256    ?    0aea3d21-13b4-452e-a177-606b1c752f1b b   
```


**5) edit all other scyllaclusters to remove the external seed**
```
$ kubectl -n scylla edit scyllaclusters.scylla.scylladb.com scylla-cluster
```

from:
```
apiVersion: scylla.scylladb.com/v1
kind: ScyllaCluster
metadata:
  name: scylla-cluster
  namespace: scylla
spec:
  agentVersion: 3.11.2
  version: 2026.2.2
  cpuset: true
  automaticOrphanedNodeCleanup: true
  exposeOptions:
    broadcastOptions:
      clients:
        type: PodIP
      nodes:
        type: PodIP
    nodeService:
      type: Headless
  externalSeeds:
 - 192.168.49.209
 - 192.168.147.149     ← node IP from DC decommissioned
```
to:
```
apiVersion: scylla.scylladb.com/v1
kind: ScyllaCluster
metadata:
  name: scylla-cluster
  namespace: scylla
spec:
  agentVersion: 3.11.2
  version: 2026.2.2
  cpuset: true
  automaticOrphanedNodeCleanup: true
  exposeOptions:
    broadcastOptions:
      clients:
        type: PodIP
      nodes:
        type: PodIP
    nodeService:
      type: Headless
  externalSeeds:
 - 192.168.49.209
```
save file and then:
```
$ kubectl -n scylla edit scyllaclusters.scylla.scylladb.com scylla-cluster

scyllacluster.scylla.scylladb.com/scylla-cluster edited
```


**6) Delete the scylla2 cluster object**

```
$ kubectl -n scylla2 delete scyllacluster scylla-cluster 

scyllacluster.scylla.scylladb.com "scylla-cluster" deleted from scylla2 namespace
```

>Note: 
May check for pv and other objects within the namespace that held resources for the decommissioned DC  
$ kubectl get all -n scylla2
$ kubectl get all --all-namespaces



finally:
```
$ kubectl --context="${CONTEXT_DC1}" -n=scylla exec -it pod/scylla-cluster-europe-west2-b-dc1-b-0 -c=scylla -- nodetool status

Datacenter: europe-west2-b-dc1
==============================
Status=Up/Down/eXcluded
|/ State=Normal/Leaving/Joining/Moving
-- Address        Load    Tokens Owns Host ID                              Rack
UN 192.168.49.209 1.25 MB 256    ?    0aea3d21-13b4-452e-a177-606b1c752f1b b   
```
