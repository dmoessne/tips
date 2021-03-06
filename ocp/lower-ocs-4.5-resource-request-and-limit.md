# use following command try to lower ocs 4.5 cpu/memory request and limit 
# see: https://rook.io/docs/rook/v1.4/ceph-cluster-crd.html

3 worker with 8 vcpu and 32 GB mem and it works in my environment.
```
# patch ocs cpu request and limit 
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"mds": {"Limit": {"cpu": "500m"}}}}}'
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"mds": {"Request": {"cpu": "500m"}}}}}'

oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"mgr": {"Limit": {"cpu": "500m"}}}}}'
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"mgr": {"Request": {"cpu": "500m"}}}}}'

oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"mon": {"Limit": {"cpu": "500m"}}}}}'
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"mon": {"Request": {"cpu": "500m"}}}}}'

oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"osd": {"Limit": {"cpu": "500m"}}}}}'
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"osd": {"Request": {"cpu": "500m"}}}}}'

oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"noobaa-core": {"Limit": {"cpu": "500m"}}}}}'
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"noobaa-core": {"Request": {"cpu": "500m"}}}}}'

oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"noobaa-db": {"Limit": {"cpu": "500m"}}}}}'
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"noobaa-db": {"Request": {"cpu": "500m"}}}}}'

oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"rgw": {"Limit": {"cpu": "500m"}}}}}'
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"rgw": {"Request": {"cpu": "500m"}}}}}'

oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"prepareosd": {"Limit": {"cpu": "500m"}}}}}'
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"prepareosd": {"Request": {"cpu": "500m"}}}}}'

oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"crashcollector": {"Limit": {"cpu": "500m"}}}}}'
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"crashcollector": {"Request": {"cpu": "500m"}}}}}'

oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"cleanup": {"Limit": {"cpu": "500m"}}}}}'
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"cleanup": {"Request": {"cpu": "500m"}}}}}'

# patch ocs memory request and limit
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"mds": {"Limit": {"memory": "512Mi"}}}}}'
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"mds": {"Request": {"memory": "512Mi"}}}}}'

oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"mgr": {"Limit": {"memory": "512Mi"}}}}}'
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"mgr": {"Request": {"memory": "512Mi"}}}}}'

oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"mon": {"Limit": {"memory": "512Mi"}}}}}'
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"mon": {"Request": {"memory": "512Mi"}}}}}'

oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"osd": {"Limit": {"memory": "512Mi"}}}}}'
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"osd": {"Request": {"memory": "512Mi"}}}}}'

oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"rgw": {"Limit": {"memory": "512Mi"}}}}}'
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"rgw": {"Request": {"memory": "512Mi"}}}}}'

oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"noobaa-db": {"Limit": {"memory": "512Mi"}}}}}'
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"noobaa-db": {"Request": {"memory": "512Mi"}}}}}'

oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"prepareosd": {"Limit": {"memory": "512Mi"}}}}}'
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"prepareosd": {"Request": {"memory": "512Mi"}}}}}'

oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"crashcollector": {"Limit": {"memory": "512Mi"}}}}}'  
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"crashcollector": {"Request": {"memory": "512Mi"}}}}}'

oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"cleanup": {"Limit": {"memory": "512Mi"}}}}}'
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"cleanup": {"Request": {"memory": "512Mi"}}}}}'
```