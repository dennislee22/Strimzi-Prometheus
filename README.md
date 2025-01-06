# Strimzi/Kafka & Prometheus+Grafana

<img width="198" alt="image" src="https://github.com/user-attachments/assets/41901918-f3d5-472a-b38e-5be2173b7d3e" />

Integrating Strimzi with Prometheus and Grafana provides a robust solution for monitoring and visualizing the health and performance of Apache Kafka clusters in Kubernetes. Strimzi, a Kubernetes Operator for managing Kafka, exposes various metrics through Prometheus exporters that can be scraped and stored by Prometheus. These metrics include Kafka broker health, throughput, and resource usage. Grafana is then used to create custom dashboards, offering visualizations of the Kafka metrics. This integration empowers DevOps teams to monitor Kafka clusters effectively, gain insights into performance bottlenecks, and ensure the system's reliability in production environments.

In this article, I will deploy the [Cloudera Streams Messaging Operator](https://docs.cloudera.com/csm-operator/1.2/index.html), which is based on Strimzi, and integrate the Kafka cluster with the existing Prometheus and Grafana platforms.

<img width="687" alt="image" src="https://github.com/user-attachments/assets/6db57df5-5158-4f6a-aa2c-64913f47b30e" />

## Deploy `Streams Messaging Operator` with Strimzi

1. Create a new namespace to host Strimzi.
```
# kubectl create ns strimzi-kafka
```

2. Create a secret object to store the Cloudera credentials.
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

5. Upon successful deployment, verify the existence of the following objects.
```
# kubectl -n strimzi-kafka get secret
NAME                                             TYPE                             DATA   AGE
csm-op-license                                   Opaque                           1      34s
sh.helm.release.v1.strimzi-cluster-operator.v1   helm.sh/release.v1               1      34s
strimzi                                          kubernetes.io/dockerconfigjson   1      7m24s
```

```
# kubectl -n strimzi-kafka get pods
NAME                                      READY   STATUS    RESTARTS   AGE
strimzi-cluster-operator-56c85645-6xswd   1/1     Running   0          41s
```

## Deploy `KafkaNodePool`
KafkaNodePool refer to groups of Kafka broker nodes that share similar configurations and resources within a Kafka cluster. By using node pools, operators can tailor resource allocation, scaling, and fault tolerance across different broker nodes, ensuring optimal performance and isolation for varying workloads or traffic patterns.

1. Create a new namespace to host the KafkaNodePool.
```
# kubectl create ns dlee-kafkanodepool
```

2. Create the KafkaNodePool by applying the `kafkanodepool.yml` file.
```
# kubectl -n dlee-kafkanodepool apply -f kafkanodepool.yml
```

3. Verify the successful creation of the pods in the Kafka pool, as shown below.
```
# kubectl -n dlee-kafkanodepool get pods
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
# kubectl -n dlee-kafkanodepool get pvc
NAME                             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-0-my-cluster-first-pool-0   Bound    pvc-3b848807-3f92-4185-a624-b946ed90ead2   100Gi      RWO            longhorn       2m57s
data-0-my-cluster-first-pool-1   Bound    pvc-246e1bc2-2f41-4323-b240-a2f2fd4e7b33   100Gi      RWO            longhorn       2m57s
data-0-my-cluster-first-pool-2   Bound    pvc-68937930-db64-42a4-8994-8367c97347bc   100Gi      RWO            longhorn       2m57s
data-my-cluster-zookeeper-0      Bound    pvc-a5198bc1-cc90-439b-b413-92aea57cc3dc   100Gi      RWO            longhorn       3m50s
data-my-cluster-zookeeper-1      Bound    pvc-d6641d3c-d833-4b8f-b473-e470b71c3619   100Gi      RWO            longhorn       3m50s
data-my-cluster-zookeeper-2      Bound    pvc-555299e5-d5f3-4a84-957a-49f07df0c6ff   100Gi      RWO            longhorn       3m50s
```

4. Run the following Kafka producer and consumer tests to verify that the Kafka cluster is operating correctly.
```
# IMAGE=$(kubectl get pod my-cluster-first-pool-0  -n dlee-kafkanodepool --output jsonpath='{.spec.containers[0].image}')
# NAMESPACE=dlee-kafkanodepool
# CLUSTER=my-cluster

# kubectl run kafka-admin -it -n $NAMESPACE --image=$IMAGE --rm=true --restart=Never --command -- /opt/kafka/bin/kafka-topics.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --create --topic anytopic1
Defaulted container "kafka-admin" out of: kafka-admin, k8tz (init)
If you don't see a command prompt, try pressing enter.
Created topic anytopic1.
pod "kafka-admin" deleted

# kubectl run kafka-producer -it -n $NAMESPACE --image=$IMAGE --rm=true --restart=Never --command -- /opt/kafka/bin/kafka-console-producer.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic anytopic1 
Defaulted container "kafka-producer" out of: kafka-producer, k8tz (init)
If you don't see a command prompt, try pressing enter.
>Dennis Lee
>is
>in the 
>house
>^C
E0106 06:06:06.531241 2809223 v2.go:104] EOF
pod "kafka-producer" deleted

# kubectl run kafka-consumer -it -n $NAMESPACE --image=$IMAGE --rm=true --restart=Never --command -- /opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic anytopic1 --from-beginning
Defaulted container "kafka-consumer" out of: kafka-consumer, k8tz (init)
If you don't see a command prompt, try pressing enter.
Dennis Lee
is
in the
house
```

## Post-Deployment Configuration

1. The Kafka cluster above is not yet configured for metrics collection. To enable this, create a ConfigMap containing the JMX metrics configuration for both Kafka and ZooKeeper.

```
# kubectl -n dlee-kafkanodepool apply -f kafka-metrics-cm.yml

# kubectl -n dlee-kafkanodepool get cm
NAME                                      DATA   AGE
kafka-metrics                             2      6h38m
kube-root-ca.crt                          1      7h2m
my-cluster-cruise-control-config          3      6h44m
my-cluster-entity-topic-operator-config   1      6h44m
my-cluster-entity-user-operator-config    1      6h44m
my-cluster-first-pool-0                   5      6h45m
my-cluster-first-pool-1                   5      6h45m
my-cluster-first-pool-2                   5      6h45m
my-cluster-zookeeper-config               3      6h46m
```

2. Edit kafka object to include the following configuration.
```
# kubectl -n dlee-kafkanodepool edit kafka

kind: Kafka
spec:
  kafka:
    metricsConfig:
      type: jmxPrometheusExporter
      valueFrom:
        configMapKeyRef:
          name: kafka-metrics
          key: kafka-metrics-config.yml
  zookeeper:
    metricsConfig:
      type: jmxPrometheusExporter
      valueFrom:
        configMapKeyRef:
          name: kafka-metrics
          key: zookeeper-metrics-config.yml
```

3. Ensure that `PodMonitor` CRD has already been created in your K8s cluster. Otherwise, check out the [Prometheus Operator](https://github.com/prometheus-operator/prometheus-operator/blob/main/Documentation/installation.md) installation procedure to install this CRD.

```
# kubectl get crds | grep podmonitor
podmonitors.monitoring.coreos.com                     2025-01-05T00:44:10Z
```

4. Apply `strimzi-pod-monitor.yml` in your existing prometheus namespace.

```
# kubectl -n infra-prometheus apply -f strimzi-pod-monitor.yml 
```

5. Verify the successful creation of the following PodMonitor objects.

```
# kubectl -n infra-prometheus get podmonitor
NAME                       AGE
bridge-metrics             3h51m
cluster-operator-metrics   3h51m
entity-operator-metrics    3h51m
kafka-resources-metrics    3h51m
```

6. Create an instance/pod of `prometheus` object for Strimzi by applying `dlee-prometheus.yml` in the existing prometheus namespace `infra-prometheus`.
```
# kubectl -n infra-prometheus get pods
NAME                                                              READY   STATUS    RESTARTS       AGE
infra-prometheus-operator-1-1736037852-kube-state-metrics-pdl88   1/1     Running   1 (35m ago)    9h
infra-prometheus-operator-1-1736037852-prometheus-node-exp42phq   1/1     Running   29 (38m ago)   9h
infra-prometheus-operator-1-1736037852-prometheus-node-exp8vrvq   1/1     Running   28 (39m ago)   9h
infra-prometheus-operator-1-1736037852-prometheus-node-expshqnk   1/1     Running   28 (37m ago)   9h
infra-prometheus-operator-1-1736037852-prometheus-node-expvmjxl   1/1     Running   4 (29m ago)    9h
infra-prometheus-operator-operator-fcc6f7f69-kp2sj                1/1     Running   1 (35m ago)    9h
prometheus-infra-prometheus-operator-prometheus-0                 2/2     Running   0              9h

# kubectl -n infra-prometheus apply -f dlee-prometheus.yml

# kubectl -n infra-prometheus get ingress strimzi-prometheus
NAME                 CLASS    HOSTS                                        ADDRESS         PORTS   AGE
strimzi-prometheus   <none>   strimzi-prometheus.apps.dlee1.cldr.example   10.129.83.133   80      4h40m

# kubectl -n infra-prometheus get pods
NAME                                                              READY   STATUS    RESTARTS       AGE
infra-prometheus-operator-1-1736037852-kube-state-metrics-pdl88   1/1     Running   1 (35m ago)    9h
infra-prometheus-operator-1-1736037852-prometheus-node-exp42phq   1/1     Running   29 (38m ago)   9h
infra-prometheus-operator-1-1736037852-prometheus-node-exp8vrvq   1/1     Running   28 (39m ago)   9h
infra-prometheus-operator-1-1736037852-prometheus-node-expshqnk   1/1     Running   28 (37m ago)   9h
infra-prometheus-operator-1-1736037852-prometheus-node-expvmjxl   1/1     Running   4 (29m ago)    9h
infra-prometheus-operator-operator-fcc6f7f69-kp2sj                1/1     Running   1 (35m ago)    9h
prometheus-infra-prometheus-operator-prometheus-0                 2/2     Running   0              9h
prometheus-strimzi-prometheus-0                                   2/2     Running   2 (3s ago)     3s

```

7. Create a new `PrometheusRule` object by applying `prometheus-rules.yml`.

```
# kubectl -n infra-prometheus apply -f prometheus-rules.yml

# kubectl -n infra-prometheus get PrometheusRule prometheus-k8s-rules
NAME                   AGE
prometheus-k8s-rules   5h28m
```

8. Subsequently, Prometheus can now scrape the Kakfa metrics. You may check out the Prometheus dashboard to verify this.
<img width="1436" alt="image" src="https://github.com/user-attachments/assets/1edf6f84-7b3d-4849-8825-78168e83b68d" />

9. Create a new Prometheus dashboard service in the K8s cluster. The internal URL endpoint of this service will later be used as the datasource in Grafana.

```
# kubectl -n infra-prometheus apply -f strimzi-prometheus-svc.yml

# kubectl -n infra-prometheus get svc strimzi-prometheus
NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)             AGE
strimzi-prometheus   ClusterIP   10.43.70.125   <none>        9090/TCP,8080/TCP   4h27m
```

10. In Grafana, configure a new datasource using the newly created Prometheus service endpoint `http://strimzi-prometheus.infra-prometheus.svc.cluster.local:9090`.
<img width="1432" alt="image" src="https://github.com/user-attachments/assets/4ce1e925-92fc-4aef-9010-22df44bd396e" />

11. The final step is import the [Grafana template for Strimzi](https://github.com/strimzi/strimzi-kafka-operator/tree/0.43.0/examples/metrics/grafana-dashboards) into Grafana and view the newly created dashboard.
<img width="1426" alt="image" src="https://github.com/user-attachments/assets/9ec19c88-3e72-4df8-bb6e-998a9d3b98ed" />

<img width="1432" alt="image" src="https://github.com/user-attachments/assets/74b5b5fd-dcf1-4658-bacc-6e3e5a0cf1a0" />





