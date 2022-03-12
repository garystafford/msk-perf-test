# Notes

All tests were run from Alpine-based Docker container running in Kubernetes Pod on Amazon EKS cluster. The EKS cluster
and MSK cluster in separate VPCs, within same AWS Account and AWS Region (us-east-1).

## Requirements

- Case 1: RF=2, acks = 1, min.insync.replicas = 1, nearest replica fetching turned on for consumer, PST not enabled
- Case 2: RF=3, acks = 1, min.insync.replicas = 1, nearest replica fetching turned on for consumer, PST not enabled
- 300-500 bytes/message (used 400)
- 50 MB/s in total
- 50 * 1024 * 1024 = 52428800 / 400 = 131072 msg/sec

## Custom MSK Configuration

```properties
auto.create.topics.enable=true
default.replication.factor=3
min.insync.replicas=1
num.io.threads=8
num.network.threads=5
num.partitions=1
num.replica.fetchers=2
replica.lag.time.max.ms=30000
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
socket.send.buffer.bytes=102400
unclean.leader.election.enable=true
zookeeper.session.timeout.ms=18000
replica.selector.class=org.apache.kafka.common.replica.RackAwareReplicaSelector
```

## Helpful Commands

```shell
# *** CHANGE ME - Bootstrap servers ***
export BOOTSTRAP_SERVERS="b-2...us-east-1.amazonaws.com:9098,b-3...us-east-1.amazonaws.com:9098,b-4...us-east-1.amazonaws.com:9098"

# Declare an array of topic names to test
declare -a TOPICS=("test_3_3" "test_50_1" "test_100_1" "test_150_1" "test_300_1")

# Number of records to produce/consume
export MESSAGE_COUNT=10000000

# Record size (Bytes)
export REC_SIZE=400
```

```shell
# create topics
bin/kafka-topics.sh \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --command-config config/client-iam.properties \
  --create \
  --topic test_3_3 \
  --partitions 3 \
  --replication-factor 3

bin/kafka-topics.sh \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --command-config config/client-iam.properties \
  --create \
  --topic test_50_1 \
  --partitions 50 \
  --replication-factor 1

bin/kafka-topics.sh \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --command-config config/client-iam.properties \
  --create \
  --topic test_100_1 \
  --partitions 100 \
  --replication-factor 1

bin/kafka-topics.sh \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --command-config config/client-iam.properties \
  --create \
  --topic test_150_1 \
  --partitions 150 \
  --replication-factor 1

bin/kafka-topics.sh \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --command-config config/client-iam.properties \
  --create \
  --topic test_300_1 \
  --partitions 300 \
  --replication-factor 1
```

```shell
# list topics
bin/kafka-topics.sh --list \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --command-config config/client-iam.properties
```

```shell
# get producer perf help
bin/kafka-producer-perf-test.sh --help

# producer perf test
for topic in ${TOPICS[@]}; do
    echo ""
    echo $topic
    bin/kafka-producer-perf-test.sh \
      --topic $topic \
      --num-records $MESSAGE_COUNT \
      --record-size $REC_SIZE \
      --throughput -1 \
      --producer.config config/producer_perf_v1.properties \
      --print-metrics
    sleep 1m
done;
```

```shell
# get consumer perf help
bin/kafka-consumer-perf-test.sh --help

# consumer perf test
for topic in ${TOPICS[@]}; do
    echo ""
    echo $topic
    bin/kafka-consumer-perf-test.sh \
      --topic $topic \
      --messages $MESSAGE_COUNT \
      --consumer.config config/consumer_perf_v1.properties \
      --bootstrap-server $BOOTSTRAP_SERVERS | \
    jq -R .|jq -sr 'map(./",")|transpose|map(join(": "))[]'
    sleep 1m
done;
```

```shell
# delete topics
for topic in ${TOPICS[@]}; do
    echo ""
    echo $topic
    bin/kafka-topics.sh --delete \
      --topic $topic \
      --bootstrap-server $BOOTSTRAP_SERVERS \
      --command-config config/client-iam.properties
done;
```

## References

- <https://docs.aws.amazon.com/msk/latest/developerguide/bestpractices.html>
- <https://docs.confluent.io/platform/current/installation/configuration/producer-configs.html>
- <https://gist.github.com/ueokande/b96eadd798fff852551b80962862bfb3>
- <https://code.amazon.com/packages/AwsSaMskKdsPerformance/trees/mainline>
- <https://medium.com/metrosystemsro/apache-kafka-how-to-test-performance-for-clients-configured-with-ssl-encryption-3356d3a0d52b>
- <https://developers.redhat.com/blog/2020/04/29/consuming-messages-from-closest-replicas-in-apache-kafka-2-4-0-and-amq-streams#using_the_rack_aware_selector_in_amq_streams>
- <https://docs.aws.amazon.com/msk/latest/developerguide/supported-kafka-versions.html>
- <https://docs.aws.amazon.com/msk/latest/developerguide/msk-configuration-properties.html>
- <https://www.confluent.io/blog/kafka-fastest-messaging-system/>
- <https://granulate.io/optimizing-kafka-performance/>

---

<i>The contents of this repository represent my viewpoints and not of my past or current employers, including Amazon Web
Services (AWS). All third-party libraries, modules, plugins, and SDKs are the property of their respective owners.</i>
