## Connect to kafka and manage access

We're going to create another service in another ns and have it try to access the recommendations-topic.
Create a new namespace for the kcat app, enable istio sidecar injection, 
```bash
kubectl create ns kcat
smm sidecar-proxy auto-inject on kcat
```

Run the following command to deploy kcat pod. The kcat application can simulate the behavior of the Producer and Consumer sending messages to Topics.

```bash
kubectl apply -n kcat -f- <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: kcat
spec:
  containers:
  - name: kafka-test
    image: "edenhill/kcat:1.7.1"
    # Just spin & wait forever
    command: [ "/bin/sh", "-c", "--" ]
    args: [ "while true; do sleep 3000; done;" ]
EOF

```
Let's first run the following command to get the Broker service of our Kafka servers.

```bash
kubectl get services -n kafka
```
Expected output:

```
NAME                          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                        AGE
kafka-0                       ClusterIP   10.104.208.233   <none>        29092/TCP,29093/TCP,9020/TCP   87m
kafka-1                       ClusterIP   10.101.37.131    <none>        29092/TCP,29093/TCP,9020/TCP   87m
kafka-all-broker              ClusterIP   10.96.228.30     <none>        29092/TCP,29093/TCP            87m
kafka-cruisecontrol-svc       ClusterIP   10.100.16.195    <none>        8090/TCP,9020/TCP              86m
kafka-kminion                 ClusterIP   10.99.224.82     <none>        8080/TCP                       87m
kafka-operator-alertmanager   ClusterIP   10.99.247.101    <none>        9001/TCP                       90m
kafka-operator-authproxy      ClusterIP   10.99.157.137    <none>        8443/TCP                       90m
kafka-operator-operator       ClusterIP   10.106.8.119     <none>        443/TCP                        90m
```
As you can see, there will be a kafka-all-broker service which binds to Kakfa pods on both port 29092 and 29093.

Next, run the following command to execute into the kcat pod.
```bash
kubectl exec -n kcat -it kcat sh
```

In our kcat pod, you can start kcat application by running the following command. We specify the flag -C to make sure kcat runs as a Consumer pulling messages from Topic recommendations-topic.
```bash
kcat -b kafka-all-broker.kafka.svc.cluster.local:29092 -C -t recommendations-topic
exit
```
(Note: **kafka-all-broker** is the K8s Service pointing our kafka instance pods). Expected output:
```
# kcat -b kafka-all-broker.kafka.svc.cluster.local:29092 -C -t recommendations-topic
% ERROR: Topic recommendations-topic error: Broker: Topic authorization failed
```

As we can see, the kcat pod is not allowed to connect and consume messages from the recommendations-topic. This is happening because we have ACL functionality in place on the kafkacluster and there is no rule to allow this traffic. Let's add a specific ACL for allowing the kcat app on the topic.

```bash
kubectl apply -f - <<EOF
apiVersion: kafka.banzaicloud.io/v1beta1
kind: KafkaACL
metadata:
  labels:
    app: kcat
  name: kcat
  namespace: kafka
spec:
  clusterRef:
    name: kafka
    namespace: kafka
  kind: User
  name: CN=kcat-default
  roles:
  - name: consumer
    resourceSelectors:
    - name: recommendations-topic
      namespace: kafka
    - name: recommendations-group
      namespace: kafka
EOF
```
Now it should be possible to access the topic and see the messages 

```bash
kubectl exec -n kcat -it kcat sh
```
```bash
kcat -b kafka-all-broker.kafka.svc.cluster.local:29092 -C -t recommendations-topic
```

The output should look like: (Ctrl+C to stop)

```
...
analytics-message
analytics-message
analytics-message
bookings-message
bookings-message
catalog-message
catalog-message
...
```

Also, you can list out the metadata for recommendations Topic by running the following command.
```bash
kcat -b kafka-all-broker.kafka.svc.cluster.local:29092 -L -t recommendations-topic
exit
```

Expected output:
```
Metadata for recommendations-topic (from broker -1: kafka-all-broker.kafka.svc.cluster.local:29092/bootstrap):
 2 brokers:
  broker 0 at kafka-0.kafka.svc.cluster.local:29092 (controller)
  broker 1 at kafka-1.kafka.svc.cluster.local:29092
 1 topics:
  topic "recommendations-topic" with 6 partitions:
    partition 0, leader 1, replicas: 1,0, isrs: 1,0
    partition 1, leader 0, replicas: 0,1, isrs: 0,1
    partition 2, leader 1, replicas: 1,0, isrs: 1,0
    partition 3, leader 0, replicas: 0,1, isrs: 0,1
    partition 4, leader 1, replicas: 1,0, isrs: 1,0
    partition 5, leader 0, replicas: 0,1, isrs: 0,1
```


To remove the ACL allowing access:

```bash
kubectl delete -f - <<EOF
apiVersion: kafka.banzaicloud.io/v1beta1
kind: KafkaACL
metadata:
  labels:
    app: kcat
  name: kcat
  namespace: kafka
spec:
  clusterRef:
    name: kafka
    namespace: kafka
  kind: User
  name: CN=kcat-default
  roles:
  - name: consumer
    resourceSelectors:
    - name: recommendations-topic
      namespace: kafka
    - name: recommendations-group
      namespace: kafka
EOF
```

If needed (to re-create later), also delete the kcat namespace:

```bash
kubectl delete namespace kcat
```



