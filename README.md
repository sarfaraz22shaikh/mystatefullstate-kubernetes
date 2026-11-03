MySQL StatefulSet on Kubernetes (kind)

A hands-on lab that deploys a 3-replica MySQL 8.0 StatefulSet with persistent storage and a headless Service on a local kind cluster.

What this demonstrates
Running a stateful workload with a Kubernetes StatefulSet
Stable pod identities (mysql-statefulset-0, -1, -2) and stable DNS names via a headless Service
Per-pod persistent storage using volumeClaimTemplates (one PVC per pod)
Ordered pod startup and how StatefulSets handle storage
Repository structure
.
├── statefullstate.yaml   # StatefulSet (3 replicas, mysql:8.0, 1Gi PVC per pod)
├── service.yaml          # Headless Service (clusterIP: None) on port 3306
└── README.md
Prerequisites
Ubuntu (VM or host) with Docker running
kind and kubectl installed
A running kind cluster (example below)
Setup
1. Create the kind cluster
bash
kind create cluster --name sarfaraz --config=config.yml
kubectl get nodes
2. Create the namespace
bash
kubectl create namespace mysql
3. Check the storage class

kind ships with a default standard StorageClass that provisions volumes dynamically.

bash
kubectl get storageclass

The StatefulSet uses storageClassName: "standard". If you use a different name, it must exist in your cluster, otherwise the PVCs stay Pending.

4. Deploy the headless Service
bash
kubectl apply -f service.yaml
kubectl get svc -n mysql

CLUSTER-IP should show None.

5. Deploy the StatefulSet
bash
kubectl apply -f statefullstate.yaml
kubectl get pods -n mysql -w

Pods start one after another: mysql-statefulset-0, then -1, then -2.

Verify
bash
kubectl get pods -n mysql
kubectl get pvc -n mysql

Expected: all pods Running, and PVCs mysql-data-mysql-statefulset-{0,1,2} in Bound state.

Connect to MySQL
bash
kubectl exec -it mysql-statefulset-0 -n mysql -- mysql -u root -p
sql
SHOW DATABASES;
Stable DNS names

Each pod is reachable at:

<pod-name>.mysql-service.mysql.svc.cluster.local

For example: mysql-statefulset-0.mysql-service.mysql.svc.cluster.local

Test persistence
bash
# Create data in pod 0
kubectl exec -it mysql-statefulset-0 -n mysql -- mysql -u root -p -e "CREATE DATABASE demo;"

# Delete the pod; the StatefulSet recreates it with the same PVC
kubectl delete pod mysql-statefulset-0 -n mysql
kubectl get pods -n mysql -w

# Check the data survived
kubectl exec -it mysql-statefulset-0 -n mysql -- mysql -u root -p -e "SHOW DATABASES;"
Troubleshooting
Problem	Cause	Fix
unknown field "spec.template.spec.containers[0].ports[0].env"	env / volumeMounts indented under ports	Make them siblings of ports under the container
unknown field "volumeClaimTemplates"	Placed at top level	Indent it under spec
unknown field "spec.clusterip"	YAML field names are case-sensitive	Use clusterIP: None
Pod stuck Pending, event: pod has unbound immediate PersistentVolumeClaims	StorageClass doesn't exist	Use standard, then delete the StatefulSet and PVCs and re-apply
pods "..." not found on kubectl exec / describe	Wrong namespace	Add -n mysql before --
Cannot connect to the Docker daemon	Docker service not running	sudo systemctl start docker and check journalctl -u docker

Tip: set a default namespace to avoid typing -n mysql each time:

bash
kubectl config set-context --current --namespace=mysql
Cleanup
bash
kubectl delete -f statefullstate.yaml
kubectl delete -f service.yaml
kubectl delete pvc -n mysql --all   # PVCs are not deleted automatically with the StatefulSet
kubectl delete namespace mysql
kind delete cluster --name sarfaraz
Security note

This lab sets MYSQL_ROOT_PASSWORD directly in the manifest for simplicity. Do not commit real passwords to GitHub. For anything beyond a local lab, store it in a Kubernetes Secret and reference it:

yaml
env:
- name: MYSQL_ROOT_PASSWORD
  valueFrom:
    secretKeyRef:
      name: mysql-secret
      key: password
bash
kubectl create secret generic mysql-secret -n mysql --from-literal=password='<your-password>'
Limitations

The 3 replicas are independent MySQL instances, each with its own storage. They are not replicating data to each other. Real MySQL replication or clustering needs extra configuration (for example an operator, or primary/replica setup with init containers).

Author

Sarfaraz
