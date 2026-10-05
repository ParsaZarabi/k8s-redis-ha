# High-Availability Redis Cluster on Kubernetes (Bare-Metal) 🚀

This repository contains the declarative Kubernetes manifests to deploy a highly available Redis cluster (1 Primary, 2 Replicas) with Redis Sentinel on a bare-metal Kubernetes environment. 

The project demonstrates the use of **StatefulSets** for stable network identities, dynamic persistent volume provisioning via **Longhorn**, and automated failover handling.

---

## 🏗 Architecture & Topology

The deployment architecture utilizes a Headless Service for internal DNS resolution and Longhorn for dynamic volume provisioning. Each Pod contains two containers: the Redis Database and the Redis Sentinel.

```mermaid
graph TD
    subgraph "Kubernetes Namespace"
        direction TB
        SVC[Headless Service <br/> redis.svc.cluster.local]

        subgraph "StatefulSet (redis)"
            direction LR
            P0[redis-0 <br/> (Primary)] 
            P1[redis-1 <br/> (Replica)] 
            P2[redis-2 <br/> (Replica)]
        end

        SVC ==> P0
        SVC ==> P1
        SVC ==> P2

        P1 -. "Data Sync" .-> P0
        P2 -. "Data Sync" .-> P0

        subgraph "Sentinel Quorum (Inside Pods)"
            S0[Sentinel-0]
            S1[Sentinel-1]
            S2[Sentinel-2]
        end

        S0 -. "Monitor & Failover" .-> P0
        S1 -. "Monitor & Failover" .-> P0
        S2 -. "Monitor & Failover" .-> P0
    end

    subgraph "Longhorn Storage (Default StorageClass)"
        V0[(PVC 1Gi <br/> redis-data-0)]
        V1[(PVC 1Gi <br/> redis-data-1)]
        V2[(PVC 1Gi <br/> redis-data-2)]
    end

    P0 --- V0
    P1 --- V1
    P2 --- V2
```

---

## ✨ Key Features & Components

*   **StatefulSet:** Guarantees ordered deployment (`redis-0` -> `1` -> `2`) and provides stable network identifiers across pod restarts.
*   **Headless Service:** Bypasses standard load balancing to allow Sentinels and replicas to communicate directly with specific Pods via DNS.
*   **InitContainers & ConfigMaps:** Dynamically bootstraps the cluster. It detects the pod's hostname (`redis-0`) to assign the Primary role, while configuring others to join as Replicas.
*   **Longhorn Integration:** Each replica automatically requests and mounts its own isolated 1Gi PersistentVolumeClaim (PVC).

---

## 🚀 Overcoming Challenges (Troubleshooting)

**Issue:** During deployment with Redis versions 8.x, the Sentinel containers entered a `CrashLoopBackOff` state with a `Failed to resolve hostname` error.  
**Root Cause:** Newer Redis Sentinel versions strictly require IP addresses and do not resolve Kubernetes hostnames by default.  
**Solution:** I configured the `InitContainer` to inject `sentinel resolve-hostnames yes` directly into the `sentinel.conf` file, and granted proper file permissions (`chmod 777`) so the Sentinel process can rewrite its own config during a failover event.

---

## 📋 Prerequisites
*   A running Kubernetes cluster (Bare-Metal or Managed).
*   **Longhorn** installed and set as the default `StorageClass`.

---

## 🛠 Deployment Instructions

1. **Deploy the ConfigMap** (Contains the initialization script and Sentinel configs):
   ```bash
   kubectl apply -f redis-config.yaml
   ```

2. **Deploy the Headless Service**:
   ```bash
   kubectl apply -f redis-svc.yaml
   ```

3. **Deploy the StatefulSet**:
   ```bash
   kubectl apply -f redis-sts.yaml
   ```

4. **Verify the Deployment**:
   ```bash
   kubectl get pods -l app=redis -w
   kubectl get pvc
   ```
   *Wait until all 3 pods reach the `2/2 Running` state.*

---

## 🔥 Testing the HA Failover

To test if the Sentinel quorum is working correctly and can handle disaster recovery:

1. Stream the logs of a Sentinel container in one terminal:
   ```bash
   kubectl logs redis-1 -c sentinel -f
   ```
2. In another terminal, forcefully delete the Primary pod to simulate a node failure:
   ```bash
   kubectl delete pod redis-0
   ```
3. Watch the Sentinel logs. After 5 seconds (`down-after-milliseconds`), you will see the Sentinels vote and automatically promote either `redis-1` or `redis-2` to be the new Primary. Once `redis-0` respawns, it will automatically join the cluster as a Replica!

---

## 💾 Testing Database Read/Write
You can verify the database functionality by executing into the pod and using `redis-cli`:
```bash
kubectl exec -it redis-0 -c redis -- redis-cli
127.0.0.1:6379> set mykey "Kubernetes HA Redis works!"
OK
127.0.0.1:6379> get mykey
"Kubernetes HA Redis works!"
```

