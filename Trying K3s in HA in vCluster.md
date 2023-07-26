# Trying K3s in HA in vCluster
The **local** Kubernetes Cluster for Rancher needs to be HA in OVH's case. At best, it should use ETCD for that.
vCluster's supported way of doing HA is by selecting an external Datastore (MYSQL, ETCD, etc.). However, OVH prefers not to provision infrastructure outside of the Host Cluster. Thus, we need to provide ETCD inside the cluster.

## Solution 1: Host a deployment of ETCD with 3 replicas
Should be possible by adding a deployment of ETCD with its own configuration for certificates.

First we need certificates for our ETCD Stateful set:
https://github.com/kelseyhightower/etcd-production-setup

For SAN, use the following:
```bash
export SAN="DNS:k3s-ha-api,DNS:k3s-ha-etcd,DNS:k3s-ha-etcd-0,DNS:k3s-ha-etcd-0.k3s-ha-etcd-headless,DNS:k3s-ha-etcd-0.k3s-ha-etcd-headless.vcluster-k3s-ha,DNS:k3s-ha-etcd-1,DNS:k3s-ha-etcd-1.k3s-ha-etcd-headless,DNS:k3s-ha-etcd-1.k3s-ha-etcd-headless.vcluster-k3s-ha,DNS:k3s-ha-etcd-2,DNS:k3s-ha-etcd-2.k3s-ha-etcd-headless,DNS:k3s-ha-etcd-2.k3s-ha-etcd-headless.vcluster-k3s-ha,DNS:k3s-ha-etcd.vcluster-k3s-ha,DNS:k3s-ha-etcd.vcluster-k3s-ha.svc,DNS:localhost,IP:0.0.0.0,IP:127.0.0.1,IP:0:0:0:0:0:0:0:1"
```

We rename the files and add the peer certificate in order to get the following hierarchy:

```bash
certs:
total 56
-rw-r--r--  1 user  staff  1883 Jul  5 13:28 etcd-ca.crt
-rw-r--r--  1 user  staff  6767 Jul  5 13:54 etcd-client.crt
-rw-r--r--  1 user  staff  6761 Jul  5 14:05 etcd-peer.crt
-rw-r--r--  1 user  staff  7317 Jul  5 13:52 etcd-server.crt

private:
total 32
-rw-r--r--  1 user  staff  3268 Jul  5 13:54 etcd-client.key
-rw-r--r--  1 user  staff  3272 Jul  5 14:04 etcd-peer.key
-rw-r--r--  1 user  staff  3272 Jul  5 13:52 etcd-server.key
-rw-r--r--  1 user  staff  3418 Jul  5 13:28 server-ca.key
```

Then, we create a secret called `k3s-ha-certs` with all these files inside:

```bash
kubectl create secret generic k3s-ha-certs -n vcluster-k3s-ha --from-file certs/etcd-ca.crt --from-file certs/etcd-client.crt --from-file certs/etcd-peer.crt --from-file certs/etcd-server.crt --from-file private/server-ca.key --from-file private/etcd-peer.key --dry-run=client -oyaml > secret-k3s-ha.yaml
```

If you are creating the secret directly in Kubernetes, you can remove the `--dry-run=client -oyaml`.

This does not seem to work as the message seems to be :

```log
Defaulted container "vcluster" out of: vcluster, syncer
time="2023-07-05T16:03:29Z" level=warning msg="Webhooks and apiserver aggregation may not function properly without an agent; please set egress-selector-mode to 'cluster' or 'pod'"
time="2023-07-05T16:03:29Z" level=info msg="Starting k3s v1.25.10+k3s1 (613a3bc8)"
```

which suggests something to do with [this](https://docs.k3s.io/installation/network-options#control-plane-egress-selector-configuration) 

## Solution 2: Make vCluster configure K3S with --init-cluster setting
Might be possible by modifying the Helm Chart for vCluster.
Probably needs to add service IPs to TLSSAN options.