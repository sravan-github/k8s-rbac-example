# Kubernetes RBAC example (Docker Desktop)

## 1. Enable Kubernetes
Docker Desktop -> Settings -> Kubernetes -> **Enable Kubernetes** -> Apply & restart.

```powershell
kubectl config use-context docker-desktop
kubectl get nodes
```

## 2. Apply the manifests
```powershell
kubectl apply -f rbac.yaml
```

Creates: namespace `dev`, ServiceAccount `dev-user`, Role `dev-role`,
RoleBinding, ClusterRole `node-reader`, ClusterRoleBinding.

## 3. Test permissions with `kubectl auth can-i` (impersonation)
```powershell
$sa = "system:serviceaccount:dev:dev-user"

kubectl auth can-i list pods        -n dev     --as $sa   # yes
kubectl auth can-i get pods/log     -n dev     --as $sa   # yes
kubectl auth can-i create deployments -n dev   --as $sa   # yes
kubectl auth can-i delete pods      -n dev     --as $sa   # no  (read-only)
kubectl auth can-i list pods        -n default --as $sa   # no  (other namespace)
kubectl auth can-i list nodes                  --as $sa   # yes (ClusterRole)
kubectl auth can-i list secrets     -n dev     --as $sa   # no
```

List everything the account can do in `dev`:
```powershell
kubectl auth can-i --list -n dev --as $sa
```

## 4. Use the ServiceAccount for real (token)
```powershell
$token = kubectl create token dev-user -n dev
$server = kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}'

kubectl --server=$server --token=$token --insecure-skip-tls-verify get pods -n dev
kubectl --server=$server --token=$token --insecure-skip-tls-verify get pods -n default   # Forbidden
```

## 5. Try creating a deployment as that user
```powershell
kubectl create deployment web --image=nginx -n dev --as $sa   # allowed
kubectl delete pod -l app=web -n dev --as $sa                 # Forbidden
```

## 6. Clean up
```powershell
kubectl delete -f rbac.yaml
```

## Key concepts
| Object | Scope | Purpose |
|---|---|---|
| Role | Namespace | Set of permissions |
| ClusterRole | Cluster | Permissions for cluster-wide or non-namespaced resources |
| RoleBinding | Namespace | Grants a Role/ClusterRole to subjects in one namespace |
| ClusterRoleBinding | Cluster | Grants a ClusterRole to subjects cluster-wide |
