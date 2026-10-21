# kind HA Cluster IP Recovery Playbook

**Environment:** Arch Linux host, Docker Engine, kind `v0.33.0`,
Kubernetes `v1.37.0`\
**Cluster:** `n8n-lab`\
**Purpose:** Diagnose and recover a kind cluster when Docker-assigned
node IPs no longer match the IP addresses embedded in etcd peer
certificates/configuration.

> **Important:** This playbook documents the recovery that worked for
> this lab. It is not a guarantee that every etcd outage is caused by IP
> reassignment. Inspect the evidence first. Do not blindly copy the
> example addresses to another cluster.

## 1. What happened

After the host restarted, the existing kind containers were assigned
different IPv4 addresses. The etcd peer certificates and static etcd
manifests still referred to the original control-plane addresses. etcd
peer TLS validation failed, quorum was lost, and the Kubernetes API
server could not connect to etcd.

The recovery was to: 1. Inspect the existing node IPs, etcd peer
certificate SANs, and etcd manifest settings. 2. Stop the existing
containers. 3. Disconnect their network endpoints. 4. Reconnect the
existing containers with explicit IPv4 addresses that matched the
original configuration. 5. Start the containers and verify all
Kubernetes nodes returned to `Ready`.

This kept the existing containers rather than deleting/recreating the
cluster.

## 2. Cluster's known-good IPv4 mapping

These are the addresses used successfully in this lab after recovery:

  Container                            IPv4 address
  ---------------------------------- --------------
  `n8n-lab-control-plane`              `172.18.0.6`
  `n8n-lab-control-plane2`             `172.18.0.5`
  `n8n-lab-control-plane3`             `172.18.0.4`
  `n8n-lab-worker`                     `172.18.0.2`
  `n8n-lab-worker2`                    `172.18.0.7`
  `n8n-lab-external-load-balancer`     `172.18.0.3`

Docker network: `kind`\
IPv4 subnet: `172.18.0.0/16`\
IPv4 gateway: `172.18.0.1`

These are **specific to this lab**. Check the actual certificates,
manifests, network, and container names before using them.

## 3. Pre-flight checks

Run from the host.

``` bash
kind get clusters
kind version
kubectl config current-context
kubectl get nodes
docker ps -a --filter "name=n8n-lab"
```

Inspect the Docker network and its attached IPv4 addresses:

``` bash
docker network inspect kind \
  --format '{{range .Containers}}{{.Name}} {{.IPv4Address}}{{"\n"}}{{end}}'

docker network inspect kind --format '{{json .IPAM.Config}}'
```

Inspect the saved network assignment for each container:

``` bash
for node in \
  n8n-lab-control-plane \
  n8n-lab-control-plane2 \
  n8n-lab-control-plane3 \
  n8n-lab-worker \
  n8n-lab-worker2 \
  n8n-lab-external-load-balancer
do
  docker inspect "$node" \
    --format '{{.Name}} IPAM={{json .NetworkSettings.Networks.kind.IPAMConfig}}'
done
```

`IPAM=null` means there is no explicit IPv4 address recorded in that
endpoint's IPAM configuration. It does not by itself prove that IP
reassignment caused an outage.

## 4. Find the IP expected by each etcd peer certificate

Run on each control-plane container:

``` bash
for node in \
  n8n-lab-control-plane \
  n8n-lab-control-plane2 \
  n8n-lab-control-plane3
do
  echo "===== $node ====="
  docker exec "$node" \
    openssl x509 \
      -in /etc/kubernetes/pki/etcd/peer.crt \
      -noout -subject -ext subjectAltName
done
```

Look for the `IP Address:` entry in `X509v3 Subject Alternative Name`.

Example from the recovered lab:

``` text
n8n-lab-control-plane   certificate IP: 172.18.0.6
n8n-lab-control-plane2  certificate IP: 172.18.0.5
n8n-lab-control-plane3  certificate IP: 172.18.0.4
```

The peer certificate may include other SANs such as `127.0.0.1`, `::1`,
and DNS names. Do not confuse those with the node's intended peer IP.

## 5. Compare with the etcd static pod manifest

Inspect the relevant flags in each control-plane node:

``` bash
for node in \
  n8n-lab-control-plane \
  n8n-lab-control-plane2 \
  n8n-lab-control-plane3
do
  echo "===== $node ====="
  docker exec "$node" sh -c \
    "grep -E -- '--(name|initial-advertise-peer-urls|initial-cluster|listen-peer-urls|advertise-client-urls|listen-client-urls)=' /etc/kubernetes/manifests/etcd.yaml"
done
```

Compare these flags with the certificate SAN and the container's current
IP:

-   `--initial-advertise-peer-urls`
-   `--listen-peer-urls`
-   `--initial-cluster`
-   Also inspect client URL flags if diagnosing broader etcd
    connectivity.

Get each current container IP:

``` bash
for node in \
  n8n-lab-control-plane \
  n8n-lab-control-plane2 \
  n8n-lab-control-plane3
do
  docker inspect "$node" \
    --format '{{.Name}} current={{.NetworkSettings.Networks.kind.IPAddress}} saved={{json .NetworkSettings.Networks.kind.IPAMConfig}}'
done
```

**Diagnosis is supported when** the current IP differs from the IP
expected by the peer certificate/configuration and etcd logs show
corresponding TLS SAN or peer-connection errors. Check etcd logs too:

``` bash
for node in \
  n8n-lab-control-plane \
  n8n-lab-control-plane2 \
  n8n-lab-control-plane3
do
  echo "===== $node ====="
  docker exec "$node" crictl ps -a --name etcd
done
```

If the etcd container is available, use its container ID from
`crictl ps -a` to inspect logs, for example:

``` bash
docker exec n8n-lab-control-plane crictl logs <ETCD_CONTAINER_ID>
```

Look for messages such as `x509: certificate is valid for ... not ...`,
`failed to verify certificate`, peer connection failures, repeated
elections, or loss of quorum.

## 6. Recovery: pin the existing containers to the expected IPs

### Safety notes

-   This procedure interrupts networking for the cluster.
-   It requires stopping containers and reattaching their Docker network
    endpoints. It is **not** a zero-interruption or "no restart"
    operation.
-   It does not delete containers, but always keep a current backup of
    application data and important configuration first.
-   Do not run `kind delete cluster`, `docker rm`,
    `docker network prune`, or delete Docker volumes as part of this
    procedure.
-   Never assign an IP already used by another endpoint.
-   Use the addresses found from your own certificates/configuration,
    not just the example mapping.
-   Static Docker IP assignment is a workaround outside kind's standard
    per-node configuration. Validate it after Docker daemon/host
    restarts.

### 6.1 Confirm the IPv4 network

``` bash
docker network inspect kind --format '{{json .IPAM.Config}}'
```

For this lab the IPv4 network was `172.18.0.0/16` with gateway
`172.18.0.1`. If your output differs, do not use the example addresses
without checking they belong to the active subnet and are free.

### 6.2 Record the mapping before making changes

``` bash
docker network inspect kind \
  --format '{{range .Containers}}{{.Name}} {{.IPv4Address}}{{"\n"}}{{end}}'
```

Write down the intended IP for every node and the load balancer. Confirm
no two containers will use the same address.

### 6.3 Stop the existing containers

In this lab, all six were stopped before network changes:

``` bash
docker stop \
  n8n-lab-external-load-balancer \
  n8n-lab-worker2 \
  n8n-lab-worker \
  n8n-lab-control-plane3 \
  n8n-lab-control-plane2 \
  n8n-lab-control-plane
```

Confirm they are stopped:

``` bash
docker ps -a --filter "name=n8n-lab"
```

### 6.4 Disconnect the network endpoints

``` bash
for node in \
  n8n-lab-external-load-balancer \
  n8n-lab-worker2 \
  n8n-lab-worker \
  n8n-lab-control-plane3 \
  n8n-lab-control-plane2 \
  n8n-lab-control-plane
do
  docker network disconnect kind "$node"
done
```

Verify the network has no attached endpoints:

``` bash
docker network inspect kind \
  --format '{{range .Containers}}{{.Name}} {{.IPv4Address}}{{"\n"}}{{end}}'
```

An empty result is expected at this point.

### 6.5 Reconnect with explicit IPs

**Example for this lab only:**

``` bash
docker network connect --ip 172.18.0.6 kind n8n-lab-control-plane
docker network connect --ip 172.18.0.5 kind n8n-lab-control-plane2
docker network connect --ip 172.18.0.4 kind n8n-lab-control-plane3

docker network connect --ip 172.18.0.2 kind n8n-lab-worker
docker network connect --ip 172.18.0.7 kind n8n-lab-worker2
docker network connect --ip 172.18.0.3 kind n8n-lab-external-load-balancer
```

These commands record explicit IPv4 assignments for the existing
containers. When containers are stopped, `docker network inspect kind`
can show no active endpoints; use `docker inspect <container>` to check
saved `IPAMConfig`. Start the containers to create active endpoints.

If any `docker network connect` command returns an error, stop and
investigate instead of continuing with a partial mapping.

### 6.6 Start the containers

In the successful recovery, the control planes were started first:

``` bash
docker start \
  n8n-lab-control-plane \
  n8n-lab-control-plane2 \
  n8n-lab-control-plane3
```

Check the network mapping:

``` bash
docker network inspect kind \
  --format '{{range .Containers}}{{.Name}} {{.IPv4Address}}{{"\n"}}{{end}}'
```

Then start the remaining containers:

``` bash
docker start \
  n8n-lab-worker \
  n8n-lab-worker2 \
  n8n-lab-external-load-balancer
```

Confirm all expected addresses are present and unique.

## 7. Validate recovery

``` bash
kubectl get nodes -o wide
kubectl get pods -A
```

All five Kubernetes nodes should return to `Ready`. Check etcd and
API-server logs if any node remains unhealthy. A successful
`kubectl get nodes` alone is not a complete etcd health test, but it is
a useful first validation.

Check that Docker retained explicit assignments:

``` bash
for node in \
  n8n-lab-control-plane \
  n8n-lab-control-plane2 \
  n8n-lab-control-plane3 \
  n8n-lab-worker \
  n8n-lab-worker2 \
  n8n-lab-external-load-balancer
do
  docker inspect "$node" \
    --format '{{.Name}} IP={{.NetworkSettings.Networks.kind.IPAddress}} IPAM={{json .NetworkSettings.Networks.kind.IPAMConfig}}'
done
```

Expected: each container shows its intended IP, and `IPAMConfig`
contains an `IPv4Address` value rather than `null`.

## 8. Test persistence before trusting a host reboot

A container restart test is a useful first check:

``` bash
docker restart n8n-lab-worker2
docker inspect n8n-lab-worker2 \
  --format 'IP={{.NetworkSettings.Networks.kind.IPAddress}} IPAM={{json .NetworkSettings.Networks.kind.IPAMConfig}}'
kubectl get nodes
```

In this lab, `n8n-lab-worker2` retained `172.18.0.7` after a container
restart. This verifies that the pinned address persisted through that
container restart.

**A container restart is not the same as a host reboot.** Before
rebooting the host, save work, ensure backups are current, and be
prepared to troubleshoot. After reboot, verify: 1.
`docker ps -a --filter "name=n8n-lab"` --- containers exist and are
running. 2. `docker network inspect kind` --- the expected IPs are
attached. 3. `kubectl get nodes -o wide` --- all nodes become `Ready`.
4. etcd logs show no peer certificate/IP mismatch.

If a host reboot changes the network or a container is recreated,
investigate why the Docker endpoint assignment was not retained. Avoid
assuming this workaround covers every Docker/kind lifecycle event.

## 9. Prevention and long-term design notes

-   Keep a copy of `kind-n8n-ha.yaml` and this playbook in version
    control.
-   Back up PostgreSQL and any persistent n8n data separately from the
    cluster configuration.
-   Do not assume Kubernetes `Node` IPs, Docker endpoint IPs, etcd peer
    certificate SANs, and etcd manifest URLs automatically update
    together; they are distinct pieces of state.
-   Do not regenerate etcd certificates or edit
    `/etc/kubernetes/manifests/etcd.yaml` casually. etcd membership is
    also persisted in its data directory, and an incomplete manual
    change can make quorum recovery harder.
-   Explicit Docker IP assignments can make this lab more predictable,
    but they are a manual workaround, not a native kind configuration
    guarantee.
-   For a more maintainable long-term lab, consider a documented
    rebuild/recovery procedure and test it periodically. For
    production-grade HA, use independent hosts or VMs with stable
    network identities; multiple kind nodes on one host do not provide
    host-level high availability.

## 10. Quick triage checklist

-   [ ] Capture `docker network inspect kind` and current container IPs.
-   [ ] Inspect etcd peer certificate SANs on each control plane.
-   [ ] Inspect etcd manifest peer URLs and initial cluster
    configuration.
-   [ ] Correlate certificate mismatch with etcd logs.
-   [ ] Confirm IPv4 subnet/gateway and ensure target addresses are
    free.
-   [ ] Back up relevant persistent data.
-   [ ] Stop existing containers before reassigning endpoints.
-   [ ] Disconnect and reconnect existing containers with explicit
    addresses.
-   [ ] Start containers and verify all IPs.
-   [ ] Confirm Kubernetes nodes are `Ready`.
-   [ ] Test container restart, then separately test a planned host
    reboot.
