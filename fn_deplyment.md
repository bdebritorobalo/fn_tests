# Federated Node Deployment on K3s - Working Instructions (2025-03-24)

## Prerequisites
- A working K3s cluster (1 master, 1+ workers)
- Helm installed and initialized
- Bitnami Helm repository added
- Federated Node Helm chart added
- Internet access from the nodes (for pulling container images)

---

## 1. Create the namespace
```bash
kubectl create namespace phems-fn-emc
```

---

## 2. Create required directories on the nodes
Ensure the following paths exist on **all nodes**:
```bash
sudo mkdir -p /mnt/storage /mnt/db /mnt/results /mnt/data
sudo chmod -R 777 /mnt/storage /mnt/db /mnt/results /mnt/data
```

---

## 3. Create the local StorageClass
```bash
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: shared-results
  annotations:
    "helm.sh/resource-policy": keep
provisioner: rancher.io/local-path
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
EOF
```

---

## 4. Create secret for PostgreSQL credentials
```bash
kubectl create secret generic db-credentials \
  --from-literal=postgres-password=adminpassword \
  --namespace phems-fn-emc
```

---

## 5. Install the PostgreSQL database using Bitnami Helm chart
```bash
helm install federated-db bitnami/postgresql \
  --namespace phems-fn-emc \
  --set auth.existingSecret=db-credentials \
  --set auth.secretKeys.adminPasswordKey=postgres-password \
  --set auth.database=federated_node_db \
  --set auth.username=admin \
  --set primary.persistence.size=8Gi
```

---

## 6. Prepare `values.yaml` for federated-node Helm chart
```yaml
ingress:
  on_aks: false
  host: "74.234.240.10.nip.io"
  nginx:
    enabled: true

db:
  host: "federated-db-postgresql.phems-fn-emc.svc.cluster.local"
  name: "federated_node_db"
  user: "admin"
  secret:
    name: "db-credentials"
    key: "postgres-password"

federatedNode:
  port: 5000
  volumes:
    results_path: "/mnt/results"
    task_pod_results_path: "/mnt/data"

storage:
  local:
    path: "/mnt/storage"
    dbpath: "/mnt/db"
```

---

## 7. Install federated-node with Helm
```bash
helm install federatednode federated-node/federated-node \
  -f values.yaml \
  --namespace=phems-fn-emc --debug
```

---

## 8. Validate installation
```bash
kubectl get pods -n phems-fn-emc
kubectl get pvc -n phems-fn-emc
kubectl get ingress -n phems-fn-emc
```

---

## 9. (Optional) Run smoketests
```bash
helm test federatednode -n phems-fn-emc --logs
```

---

## Notes:
- If you get `PVC Pending` errors, check that the StorageClass is correct and the paths exist on the node.
- If you reinstall, you may need to manually delete the existing `PV`, `PVC`, and `StorageClass`.
- To uninstall everything:
```bash
helm uninstall federatednode -n phems-fn-emc
helm uninstall federated-db -n phems-fn-emc
kubectl delete pvc --all -n phems-fn-emc
kubectl delete pv flask-results-pv || true
kubectl delete sc shared-results || true
kubectl delete ns phems-fn-emc
```

---

✅ Deployment complete and verified working as of 2025-03-24
