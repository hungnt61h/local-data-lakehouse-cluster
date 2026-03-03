# Data Lakehouse Local Cluster
*Vibe data lakehouse*

## I. Physical Machines & Devices
### 1. Machine 1
- CPU: 16 cores
- Memory: 32 GB
- Storage: 500 GB
- Ethernet: 2.5 Gbps
### 2. Machine 2
- CPU: 32 cores
- Memory: 128 GB
- Storage: 500 GB
- WiFi 6 connection
### 3. Router
- TP Link AX 53 with DHCP `192.168.0.0/24`
- Reserve IP addresses from ranges: `192.168.0.29 - 192.168.0.50` and `192.168.0.200 - 192.168.0.250`

## II. Virtual Machines
*All of them run Centos 10 Stream on VMWare Workstation Pro 17.5+*
### 1. Physical Machine 1

| Name  | Type          | IP           | FQDN         | CPUs | Memory | Storage |
|-------|---------------|--------------|--------------|------|--------|---------|
| node6 | Control Plane | 192.168.0.35 | node6.dp.com | 2    | 8 GB   | 50 GB   |
| node7 | Worker        | 192.168.0.36 | node7.dp.com | 12   | 16 GB  | 200 GB  |

### 2. Physical Machine 2

| Name  | Type          | IP           | FQDN         | CPUs | Memory | Storage |
|-------|---------------|--------------|--------------|------|--------|---------|
| node1 | Control Plane | 192.168.0.30 | node1.dp.com | 2    | 8 GB   | 50 GB   |
| node2 | Control Plane | 192.168.0.31 | node2.dp.com | 2    | 8 GB   | 50 GB   |
| node3 | Worker        | 192.168.0.32 | node3.dp.com | 8    | 32 GB  | 200 GB  |
| node4 | Worker        | 192.168.0.33 | node4.dp.com | 8    | 32 GB  | 200 GB  |
| node5 | Worker        | 192.168.0.34 | node5.dp.com | 8    | 32 GB  | 200 GB  |

## III. Architecture
Below is the desired architecture, however, it's overkilled for a lab environment. Therefore, only core components are installed (ignore authentication, authorization, and monitoring/alerts).

![Alt Text](./architecture.gif)

## IV. Installation
### 1. Services
1. MetalLB Pool
2. HAProxy Ingress Controller
3. cert-manager
4. MinIO
5. Kafka (Apache Kafka, KRaft mode)
6. Flink (Flink Operator)
7. Spark (nothing to install, just need `spark-submit` on the client side)
8. StarRocks
9. Gravitino + Gravitino Iceberg REST Catalog Server (MySQL metadata storage)
10. Airflow (Airflow 3.1.7, Kubernetes executor, MinIO DAG/log storage)
11. Metabase

### 2. Install K8s and configure OS with Ansible
*Before doing the steps below, install `ansible` first ([guide](https://docs.ansible.com/projects/ansible/latest/installation_guide/intro_installation.html))*

1. Prepare ansible vault for ssh an sudo password
```shell
ansible-vault encrypt_string --ask-vault-pass '<password>' --name 'password'
```
2. Paste the output content to `ansible_password` and `ansible_become_password` in the `all.yaml` file
3. Run the following commands to configure OS and install K8s
```shell
# From the repo directory
cd ansible

ansible-playbook -i ./inventory install_prereqs.yml --ask-vault-pass
ansible-playbook -i ./inventory update_firewall.yml --ask-vault-pass
ansible-playbook -i ./inventory install_kubernetes.yml --ask-vault-pass
ansible-playbook -i ./inventory install_kubeadm_ha.yml --ask-vault-pass
ansible-playbook -i ./inventory install_calico.yml --ask-vault-pass
ansible-playbook -i ./inventory install_metallb.yml --ask-vault-pass
ansible-playbook -i ./inventory install_haproxy_ingress.yml --ask-vault-pass
ansible-playbook -i ./inventory install_metrics_server.yml --ask-vault-pass
ansible-playbook -i ./inventory install_headlamp.yml --ask-vault-pass
ansible-playbook -i ./inventory install_cert_manager.yml --ask-vault-pass
ansible-playbook -i ./inventory install_minio.yml --ask-vault-pass
ansible-playbook -i ./inventory install_kafka.yml --ask-vault-pass
ansible-playbook -i ./inventory install_starrocks.yml --ask-vault-pass
ansible-playbook -i ./inventory install_flink_operator.yml --ask-vault-pass
ansible-playbook -i ./inventory install_gravitino.yml --ask-vault-pass
ansible-playbook -i ./inventory install_airflow.yml --ask-vault-pass
ansible-playbook -i ./inventory install_metabase.yml --ask-vault-pass
```

4. Optional firewall tuning for direct Flink/Spark host ports
- Default rules already cover in-cluster Flink/Spark traffic (`pod_cidr`, `service_cidr`, and NodePort TCP range)
- If you expose Flink/Spark ports directly on nodes, set `firewalld_enable_flink_spark_direct_ports: true` in `ansible/inventory/group_vars/all.yaml`, then rerun:
```shell
ansible-playbook -i ./inventory update_firewall.yml --ask-vault-pass
```

5. MinIO distributed mode notes
- The MinIO role is configured in distributed mode by default and schedules MinIO pods on worker nodes
- MinIO role auto-installs `local-path` StorageClass (local node disk) when the configured storage class is missing
- If old MinIO PVCs were created without storage class, run MinIO playbook once with `-e minio_force_reinstall_on_storage_class_mismatch=true`

6. Apache Kafka (KRaft) notes
- Kafka role deploys Apache Kafka `4.2.0` in KRaft mode with 3 brokers and persistent PVCs (no MinIO dependency)
- Kafka role also deploys Confluent Schema Registry in namespace `kafka`
- External broker ports `9092-9094` are routed by HAProxy TCP config
- Schema Registry is exposed through HAProxy TCP on `http://schema-registry.<your-domain>:8081`
- Schema Registry ingress host is also available at `http://schema-registry.<your-domain>`
- Schema Registry pod disables Kubernetes service-link env injection to avoid deprecated `PORT` startup failure
- Apply updated firewall and hosts mappings:
```shell
ansible-playbook -i ./inventory update_firewall.yml --ask-vault-pass
```
- If HAProxy was installed before adding Kafka TCP config, rerun:
```shell
ansible-playbook -i ./inventory install_haproxy_ingress.yml --ask-vault-pass
```

7. StarRocks (shared-data) notes
- StarRocks role deploys StarRocks shared-data mode via `kube-starrocks` chart with `3` FE and `3` CN replicas
- FE/CN images default to StarRocks `4.0-latest` (`starrocks/fe-ubuntu` and `starrocks/cn-ubuntu`)
- FE pods request/limit `2` CPU and `4Gi` memory, and use a `10Gi` PVC (`local-path` by default) for FE metadata
- CN pods request/limit `4` CPU and `8Gi` memory
- FE is configured to use MinIO as S3-compatible shared storage
- Role bootstraps MinIO bucket `starrocks` by default before installing the cluster
- FE HTTP is exposed via ingress host `http://starrocks.<your-domain>`
- FE MySQL/SQL port `9030` is added to HAProxy TCP services config map
- If HAProxy was installed before adding StarRocks TCP port, rerun:
```shell
ansible-playbook -i ./inventory install_haproxy_ingress.yml --ask-vault-pass
```
- Apply updated firewall and hosts mappings:
```shell
ansible-playbook -i ./inventory update_firewall.yml --ask-vault-pass
```

8. Apache Gravitino notes
- Gravitino role deploys Apache Gravitino from the official source chart at Git tag `v1.1.0` (`dev/charts/gravitino`)
- Default settings use `simple` authenticator, one replica, and relational entity store on embedded H2 (`jdbc:h2`) with PVC persistence enabled
- This repo overrides metadata backend to a dedicated MySQL StatefulSet (`gravitino-mysql`) with `5Gi` PVC in namespace `gravitino`
- Role auto-initializes Gravitino MySQL metadata schema before starting the Gravitino server
- Gravitino is wired to MinIO credentials for Iceberg S3 I/O and defaults the warehouse to `s3://gravitino/iceberg/`
- Default MinIO bootstrap auto-creates the bucket `gravitino` (`gravitino_minio_auto_create_bucket: true`)
- Ingress defaults to `http://gravitino.<your-domain>` using class `haproxy`
- The same role also deploys standalone Apache Gravitino Iceberg REST Catalog Server (`gravitino-iceberg-rest-server`) with ingress `http://gravitino-iceberg-rest.<your-domain>`
- Standalone Iceberg REST server chart defaults to static `memory` catalog backend; this repo overrides to `jdbc` on the same MySQL instance as Gravitino metadata
- For Gravitino-managed catalogs configure `gravitino_iceberg_rest_catalog_config_provider` and dynamic provider variables in role defaults/group vars
- If you changed `services`, ingress host/IP, or firewall variables, rerun:
```shell
ansible-playbook -i ./inventory update_firewall.yml --ask-vault-pass
ansible-playbook -i ./inventory install_haproxy_ingress.yml --ask-vault-pass
```

9. Apache Airflow notes
- Airflow role deploys Airflow `3.1.7` via Helm chart `apache-airflow/airflow` `1.19.0`
- Topology defaults: 1 API/web endpoint (`apiServer`), 2 schedulers, 4 Celery workers
- Executor defaults to `KubernetesExecutor` so Kubernetes execution is enabled while dedicated workers are available *(to save time, just build your own Airflow image with pre-installed packages you need and set it to helm values)*
- DAG bundles are sourced from MinIO (`airflow-dags` bucket). Remote logs are stored in MinIO (`airflow-logs` bucket)
- Keep MinIO remote logs enabled with `airflow_remote_logging_enabled: true`
- MinIO DAG bundle defaults use `airflow_minio_dags_bucket: airflow-dags` and `airflow_minio_dags_prefix: dags`.
- Triggerer log PVC is disabled by default (`airflow_triggerer_persistence_enabled: false`)
- Internal PostgreSQL uses static local PV/PVC (5Gi by default)
- Access Airflow UI/API at `http://airflow.<your-domain>`
- If you changed `services`, ingress host/IP, or firewall variables, rerun:
```shell
ansible-playbook -i ./inventory update_firewall.yml --ask-vault-pass
ansible-playbook -i ./inventory install_haproxy_ingress.yml --ask-vault-pass
```

10. Metabase notes
- Metabase role deploys Helm chart `pmint93/metabase` in namespace `metabase`
- Role deploys an internal PostgreSQL metadata database (`metabase-postgresql`) in the same namespace
- PostgreSQL metadata storage uses a `5Gi` PVC (`local-path` by default)
- Ingress defaults to `http://metabase.<your-domain>` using class `haproxy`
- Bootstrap job initializes admin credentials `admin@<domain>/admin` (Metabase login email defaults to `admin@dp.com`)
- Password policy defaults are set to `weak` with minimum length `5` so the bootstrap password `admin` is accepted
- `/etc/hosts` mapping includes `metabase.<your-domain>` through `services` + `update_firewall` role
- No additional firewall/Haproxy TCP ports are required (Metabase is served via ingress on existing `80/443`)
- If you changed `services`, ingress host/IP, or firewall variables, rerun:
```shell
ansible-playbook -i ./inventory update_firewall.yml --ask-vault-pass
ansible-playbook -i ./inventory install_haproxy_ingress.yml --ask-vault-pass
```
