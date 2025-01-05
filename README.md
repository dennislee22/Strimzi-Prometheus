# Strimzi & Prometheus+Grafana

<img width="198" alt="image" src="https://github.com/user-attachments/assets/41901918-f3d5-472a-b38e-5be2173b7d3e" />

## Deploy `Streams Messaging Operator`

1. Create a new namespace to host Strimzi.
```
# kubectl create ns strimzi-kafka
```

```
# kubectl create secret docker-registry strimzi --docker-server container.repository.cloudera.com --docker-username xxx --docker-password yyy --namespace strimzi-kafka
```

```
# kubectl -n strimzi-kafka get secret
NAME                                             TYPE                             DATA   AGE
csm-op-license                                   Opaque                           1      34s
sh.helm.release.v1.strimzi-cluster-operator.v1   helm.sh/release.v1               1      34s
strimzi                                          kubernetes.io/dockerconfigjson   1      7m24s
```

```
# helm install strimzi-cluster-operator --namespace strimzi-kafka --set 'image.imagePullSecrets[0].name=strimzi' --set-file clouderaLicense.fileContent=/root/license.txt --set watchAnyNamespace=true oci://container.repository.cloudera.com/cloudera-helm/csm-operator/strimzi-kafka-operator --version 1.2.0-b54
```




```
# kubectl -n strimzi-kafka get pods`
NAME                                      READY   STATUS    RESTARTS   AGE
strimzi-cluster-operator-56c85645-6xswd   1/1     Running   0          41s
```

```
# kubectl create ns dlee-kafkanodepool
```

<img width="1432" alt="image" src="https://github.com/user-attachments/assets/74b5b5fd-dcf1-4658-bacc-6e3e5a0cf1a0" />

https://github.com/strimzi/strimzi-kafka-operator/tree/0.43.0/examples/metrics/grafana-dashboards

<img width="1436" alt="image" src="https://github.com/user-attachments/assets/1edf6f84-7b3d-4849-8825-78168e83b68d" />

