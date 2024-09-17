# Kafka Migration

## Create Kafka `kafka-dev`

```
kubectl create -f kafka-dev.yaml
kafka.kubedb.com/kafka-dev created
```
## Create a topic named `test`

```bash
kubectl exec -it kafka-dev-0 -n demo -- kafka-topics.sh --bootstrap-server localhost:9092 --create --topic test --partitions 1 --replication-factor 1
Created topic test.
```

## Produce messages to the topic `test`

```bash
kubectl exec -it kafka-dev-0 -n demo -- kafka-console-producer.sh --broker-list localhost:9092 --topic test
> abc
> check
> 123
```

## Consume messages from the topic `test`

```bash
kubectl exec -it kafka-dev-0 -n demo -- kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning
abc
check
123
```

## Create another topic named `migration`

```bash
kubectl exec -it kafka-dev-0 -n demo -- kafka-topics.sh --bootstrap-server localhost:9092 --create --topic migration --partitions 1 --replication-factor 1
Created topic migration.
```

## Create producer and consumer for the topic `migration`

```bash
kubectl create -f producer-old.yaml
job.batch/producer-old created

kubectl create -f consumers-old.yaml
job.batch/consumer-old created
```

## Check the logs of the producer and consumer

## Create a new Kafka cluster `kafka-prod` and Kafka Connect cluster `connect-cluster`

```bash
kubectl create -f kafka-prod.yaml
kafka.kubedb.com/kafka-prod created

kubectl create -f connect-cluster.yaml
connectcluster.kafka.kubedb.com/connect-cluster created
```

## Wait for the Kafka cluster `kafka-prod` and Kafka Connect cluster `connect-cluster` to be ready

## Create Mirror source, checkpoint and heartbeats connectors

```bash
kubectl create -f connectors.yaml
secret/mirror-maker-source-config created
secret/mirror-maker-checkpoint-config created
secret/mirror-maker-heartbeat-config created
connector.kafka.kubedb.com/mirror-source-connector created
connector.kafka.kubedb.com/mirror-checkpoint created
connector.kafka.kubedb.com/mirror-heartbeat created
```

## Check the status of the connectors

```bash
kubectl get connector -n demo
NAME                                TYPE                        CONNECTCLUSTER    STATUS    AGE
mirror-checkpoint                   kafka.kubedb.com/v1alpha1   connect-cluster   Running   20s
mirror-heartbeat                    kafka.kubedb.com/v1alpha1   connect-cluster   Running   20s
mirror-source-connector             kafka.kubedb.com/v1alpha1   connect-cluster   Running   20s

```

## Move the consumer first to the new Kafka cluster `kafka-prod`

Before moving the consumer, we need to check the difference of consumer-group `consumer lag` in the old Kafka cluster `kafka-dev` with the new Kafka cluster `kafka-prod`. If the difference is minimal then we will stop the consumer in the old Kafka cluster and wait for the consumer-group to be fully synced in the new Kafka cluster.

## Check the consumer group current offset in the old Kafka cluster `kafka-dev` and new Kafka cluster `kafka-prod`

```bash
$ kubectl exec -it -n demo kafka-dev-0 -- kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --all-groups --offsets

GROUP             TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                 HOST            CLIENT-ID
group-migration-1 migration       0          3686594         3686594         0               sarama-421f0b47-e27d-49eb-bd44-2d9508cdc3ad /10.2.0.66      sarama

GROUP           TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                 HOST            CLIENT-ID
group-test-1    test            0          16              16              0               sarama-c49a74c8-3e64-4fa8-84ac-49fa2c2c5a61 /10.2.0.67      sarama
```

and

```bash
$ kubectl exec -it -n demo kafka-prod-broker-0 -- kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --all-groups --offsets

Consumer group 'group-migration-1' has no active members.

GROUP             TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID     HOST            CLIENT-ID
group-migration-1 migration       0          3686594         3686594         0               -               -               -

Consumer group 'group-test-1' has no active members.

GROUP           TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID     HOST            CLIENT-ID
group-test-1    test            0          16              16              0               -               -               -
```

Both consumer-groups are in sync.

### Delete the consumer in the old Kafka cluster `kafka-dev`

```bash
kubectl delete -f consumers-old.yaml
job.batch "consumer-old" deleted
```
## Create a new consumer in the new Kafka cluster `kafka-prod`

```bash
kubectl create -f consumers-new.yaml
job.batch/consumer-new created
```
## Check all topics partitions are in minimal difference in the new Kafka cluster `kafka-prod`

## Stop the producer in the old Kafka cluster `kafka-dev`

```bash
kubectl delete -f producer-old.yaml
job.batch "producer-old" deleted
```

Wait for all topics partitions to be synced in the new Kafka cluster `kafka-prod`

## Create a new producer in the new Kafka cluster `kafka-prod`

```bash
kubectl create -f producer-new.yaml
job.batch/producer-new created
```

## Remove all mirror-maker connectors

```bash
kubectl delete -f connectors.yaml
secret "mirror-maker-source-config" deleted
secret "mirror-maker-checkpoint-config" deleted
secret "mirror-maker-heartbeat-config" deleted
connector.kafka.kubedb.com "mirror-source-connector" deleted
connector.kafka.kubedb.com "mirror-checkpoint" deleted
connector.kafka.kubedb.com "mirror-heartbeat" deleted
```

## Delete old Kafka cluster `kafka-dev`

```bash
kubectl delete -f kafka-dev.yaml
kafka.kubedb.com "kafka-dev" deleted
```

Migration is completed successfully.







