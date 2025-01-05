# Strimzi & Prometheus+Grafana

<img width="198" alt="image" src="https://github.com/user-attachments/assets/41901918-f3d5-472a-b38e-5be2173b7d3e" />

Integrating Strimzi with Prometheus and Grafana provides a robust solution for monitoring and visualizing the health and performance of Apache Kafka clusters in Kubernetes. Strimzi, a Kubernetes Operator for managing Kafka, exposes various metrics through Prometheus exporters that can be scraped and stored by Prometheus. These metrics include Kafka broker health, throughput, and resource usage. Grafana is then used to create custom dashboards, offering visualizations of the Kafka metrics. This integration empowers DevOps teams to monitor Kafka clusters effectively, gain insights into performance bottlenecks, and ensure the system's reliability in production environments.

In this article, I will deploy the Cloudera Streams Messaging Operator, which is based on Strimzi, and integrate the Kafka cluster with the existing Prometheus and Grafana platforms.

## Deploy `Streams Messaging Operator` with Strimzi

1. Create a new namespace to host Strimzi.
```
# kubectl create ns strimzi-kafka
```

2. Create a secret object to store the Cloudera credentials. This secret object must exist in the namespace where you install Strimzi as well as all namespaces where you deploy Kafka or Kafka Connect clusters.   
```
# kubectl create secret docker-registry strimzi --docker-server container.repository.cloudera.com --docker-username xxx --docker-password yyy --namespace strimzi-kafka
```

3. Log in to the Cloudera Docker registry using helm chart. 
```
# helm registry login container.repository.cloudera.com
```

4. Deploy Strimzi using helm.
```
# helm install strimzi-cluster-operator --namespace strimzi-kafka --set 'image.imagePullSecrets[0].name=strimzi' --set-file clouderaLicense.fileContent=/root/license.txt --set watchAnyNamespace=true oci://container.repository.cloudera.com/cloudera-helm/csm-operator/strimzi-kafka-operator --version 1.2.0-b54
```

5. Upon successful deployment, verify the existence of the following objects
```
# kubectl -n strimzi-kafka get secret
NAME                                             TYPE                             DATA   AGE
csm-op-license                                   Opaque                           1      34s
sh.helm.release.v1.strimzi-cluster-operator.v1   helm.sh/release.v1               1      34s
strimzi                                          kubernetes.io/dockerconfigjson   1      7m24s
```

```
# kubectl -n strimzi-kafka get pods`
NAME                                      READY   STATUS    RESTARTS   AGE
strimzi-cluster-operator-56c85645-6xswd   1/1     Running   0          41s
```

## Deploy `Kafka Node Pool`
Kafka node pools refer to groups of Kafka broker nodes that share similar configurations and resources within a Kafka cluster. By using node pools, operators can tailor resource allocation, scaling, and fault tolerance across different broker nodes, ensuring optimal performance and isolation for varying workloads or traffic patterns.

1. Create a new namespace.
```
# kubectl create ns dlee-kafkanodepool
```

https://docs.cloudera.com/csm-operator/1.2/kafka-deploy-configure/topics/csm-op-deploying-kafka.html#concept_oks_bgz_z1c

```
oc -n dlee-kafkanodepool apply -f kafkanodepool.yml
```

```
oc -n dlee-kafkanodepool get pods
NAME                                          READY   STATUS    RESTARTS   AGE
my-cluster-cruise-control-874d98876-6zwt2     1/1     Running   0          93s
my-cluster-entity-operator-86d4659f96-rpbt8   2/2     Running   0          114s
my-cluster-first-pool-0                       1/1     Running   0          2m45s
my-cluster-first-pool-1                       1/1     Running   0          2m45s
my-cluster-first-pool-2                       1/1     Running   0          2m44s
my-cluster-zookeeper-0                        1/1     Running   0          3m38s
my-cluster-zookeeper-1                        1/1     Running   0          3m38s
my-cluster-zookeeper-2                        1/1     Running   0          3m38s
```

```
oc -n dlee-kafkanodepool get pvc
NAME                             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-0-my-cluster-first-pool-0   Bound    pvc-3b848807-3f92-4185-a624-b946ed90ead2   100Gi      RWO            longhorn       2m57s
data-0-my-cluster-first-pool-1   Bound    pvc-246e1bc2-2f41-4323-b240-a2f2fd4e7b33   100Gi      RWO            longhorn       2m57s
data-0-my-cluster-first-pool-2   Bound    pvc-68937930-db64-42a4-8994-8367c97347bc   100Gi      RWO            longhorn       2m57s
data-my-cluster-zookeeper-0      Bound    pvc-a5198bc1-cc90-439b-b413-92aea57cc3dc   100Gi      RWO            longhorn       3m50s
data-my-cluster-zookeeper-1      Bound    pvc-d6641d3c-d833-4b8f-b473-e470b71c3619   100Gi      RWO            longhorn       3m50s
data-my-cluster-zookeeper-2      Bound    pvc-555299e5-d5f3-4a84-957a-49f07df0c6ff   100Gi      RWO            longhorn       3m50s
```

<img width="1432" alt="image" src="https://github.com/user-attachments/assets/74b5b5fd-dcf1-4658-bacc-6e3e5a0cf1a0" />

https://github.com/strimzi/strimzi-kafka-operator/tree/0.43.0/examples/metrics/grafana-dashboards

<img width="1436" alt="image" src="https://github.com/user-attachments/assets/1edf6f84-7b3d-4849-8825-78168e83b68d" />

