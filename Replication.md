# Replication Notes

- Replicates between Datacenters, Racks, and VolumeServers
- Datalocality can choose closest datacenter

## Requirements

- Nodes need to be labeled

```
kubectl label nodes scremlin rack=main --overwrite
kubectl label nodes scremlin dataCenter=home --overwrite
kubectl label nodes scremlin topology.kubernetes.io/zone=home --overwrite
kubectl label nodes scremlin topology.kubernetes.io/region=home --overwrite

kubectl label nodes bradley rack=main --overwrite
kubectl label nodes bradley dataCenter=home --overwrite
kubectl label nodes bradley topology.kubernetes.io/zone=home --overwrite
kubectl label nodes bradley topology.kubernetes.io/region=home --overwrite
```

- Kubemod needs to be installed: https://github.com/kubemod/kubemod

## Test results

- Proxy vs Local setting in swfs values doesnt have an effect
- max speed 440/220 seems limited by speed of replica drives on Bradley (raid1)
- Will re-test with using SSD's on both machines
