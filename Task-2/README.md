# Task 2: FastAPI + MySQL with ConfigMap, Secret & Ingress

## Environment and Differences from the Task Description

This cluster is not the minikube cluster the task assumes. The differences below explain every place where my results deviate from the original instructions.

| Topic | Task description | My environment |
|-------|------------------|----------------|
| Cluster | minikube, 3 nodes (Task 1) | **kubeadm**, 3 nodes: `server`, `node-0`, `node-1` |
| Network | Internet access | **Air-gapped** (no external access) |
| PV node | `minikube-m02` | **`node-0`** |
| Image loading | `minikube image load` | Images copied to every node with **Ansible** |
| Ingress ADDRESS | Filled | **Empty** (see "Differences from the Expected Results") |

---

## Task Structure

The folder contains only the Kubernetes resources, grouped by component. The provided source files (the app code and the MySQL image files) are not included.

- **`webapp/`** contains all the resources that belong to the web app, with the name `webapp-*` (for example the ConfigMap `webapp-config`, the Deployment and Service `webapp-api`, and the Ingress `webapp-ingress`).
- **`mysql/`** contains all the resources that belong to the database, with the name `mysql-*` (for example the Secret `mysql-secret`, the PV `mysql-pv`, the PVC `mysql-pvc`, and the Deployment and Service `mysql`).
- **`screenshots/`** contains the screenshots of every verification, named `02-<name>.png`.

```
Task-2/
├── README.md
├── webapp/        # webapp-* resources
├── mysql/         # mysql-* resources
└── screenshots/   # XX-<name>.png
```

---

## Verification

### 1. Images loaded

**Not verifiable.** The cluster uses kubeadm and no minikube, so there is no `minikube image ls` output to show. The images were copied to all nodes (`server`, `node-0`, `node-1`) with Ansible. The Deployments use `imagePullPolicy: IfNotPresent`, so a successful Pod start without `ErrImagePull`/`ImagePullBackOff` (Checks 4 and 5) confirms the images are present on the nodes.

### 2. Secret and ConfigMap

Expected: DATA `3` and `5`.

![Secret and ConfigMap](screenshots/02-secret-and-configmap.png)

### 3. PV and PVC bound

Expected: `Bound`, `2Gi`.

![PV and PVC bound](screenshots/03-pv-and-pvc-bound.png)

### 4. Deployments ready

Expected: `mysql 1/1`, `webapp-api 2/2`.

![Deployments ready](screenshots/04-deployments-ready.png)

### 5. MySQL on the PV node

Expected: `1/1 Running`, NODE `node-0` (instead of `minikube-m02`).

![MySQL on the PV node](screenshots/05-mysql-on-the-pv-node.png)

### 6. App env

Expected: `APP_TITLE`, `MYSQL_HOST=mysql`, `MYSQL_USER`, `MYSQL_PASSWORD`.

![App env](screenshots/06-app-env.png)

### 7. Default data

Expected: the 2 default messages in `webapp.messages`.

![Default data](screenshots/07-default-data.png)

### 8. Ingress

Expected: CLASS `nginx`, ADDRESS filled. **ADDRESS is empty here**, see "Differences from the Expected Results".

![Ingress](screenshots/08-ingress.png)

### 9. App info

Expected: `/api/info` returns `version: task2`, the title, `environment: dev`, and `db`.

![App info](screenshots/09-app-info.png)

### 10. Write a message

Expected: `{"id": 3, "written_to": "mysql-..."}`.

![Write a message](screenshots/10-write-a-message.png)

### 11. Read messages

Expected: my message plus the 2 default ones.

![Read messages](screenshots/11-read-messages.png)

### 12. Data survives a restart

Expected: the message is still there after deleting the MySQL Pod.

![Data survives a restart](screenshots/12-data-survives-a-restart.png)

---

## Differences from the Expected Results

### Check 1: images not verifiable with `minikube image ls`
See Check 1 above. The images were distributed with Ansible, not `minikube image load`.

### Check 8: Ingress ADDRESS is empty
The Ingress ADDRESS is filled from the status of the Ingress Controller's LoadBalancer Service (its external IP). In this air-gapped cluster there is no cloud load balancer, and the nodes have no external IP, so no address is ever assigned. This does not affect routing: the Ingress itself works.

### `minikube-m02` replaced by `node-0`
Everywhere the task mentions `minikube-m02` (the PV `nodeAffinity` and Check 5), I used `node-0`.

---

## Problems and Fixes

### MySQL initdb script failed because of lost file permissions
The provided `mysql/` folder was downloaded on a Windows machine. Windows does not keep the Unix executable bit, so `initdb/01-setup.sh` was copied into the image without correct permission and MySQL's first-start initialization broke. I fixed it by **editing the provided MySQL `Dockerfile`** to set the permission explicitly inside the image:

<!-- TODO: replace this snippet with the exact line you added -->
```dockerfile
FROM mysql:8.4

# Default database and application user.
# Override any of them with environment variables at runtime.
# MYSQL_ROOT_PASSWORD has no default: it must be given at runtime.
ENV APP_DB_NAME=webapp \
    APP_DB_USER=webapp \
    APP_DB_PASSWORD=webapp123 \
    MYSQL_INITDB_SKIP_TZINFO=1

# Server settings
COPY --chmod=0644 conf/mysql.cnf /etc/mysql/conf.d/mysql.cnf

# Tables and default data used by the app
COPY --chmod=0644 schema/schema.sql /opt/webapp/schema.sql
RUN chmod 0755 /opt/webapp

# Runs once, on the first start with an empty data directory
COPY --chmod=0644 initdb/01-setup.sh /docker-entrypoint-initdb.d/01-setup.sh
```

This is the only change made to the provided files. The files themselves are not part of this folder.

---

## Summary

All components of the stack run in `webapp-stack`: 1 MySQL Pod on `node-0` with persistent storage, 2 app Pods, a ConfigMap, a Secret, and an Ingress. Data written through the app survives a MySQL Pod restart. The only deviations from the task are caused by the kubeadm, air-gapped environment, and are documented above.