# Notes

- Case 1: RF=2, acks = 1, min.insync.replicas = 1, nearest replica fetching turned on for consumer, PST not enabled
- Case 2: RF=3, acks = 1, min.insync.replicas = 1, nearest replica fetching turned on for consumer, PST not enabled
- 300-500 bytes/message (used 400)
- 50 MB/s in total
- 50 * 1024 * 1024 = 52428800 / 400 = 131072 msg/sec

## Helpful Commands

```shell
# *** CHANGE ME ***
export BBROKERS="b-3.change-me...kafka.us-east-1.amazonaws.com:9098,b-1.change-me...kafka.us-east-1.amazonaws.com:9098,b-4.change-me...kafka.us-east-1.amazonaws.com:9098"

# create topics
bin/kafka-topics.sh \
  --bootstrap-server $BBROKERS \
  --command-config config/client-iam.properties \
  --create \
  --topic test_3_3 \
  --partitions 3 \
  --replication-factor 3

bin/kafka-topics.sh \
  --bootstrap-server $BBROKERS \
  --command-config config/client-iam.properties \
  --create \
  --topic test_150_1 \
  --partitions 150 \
  --replication-factor 1

bin/kafka-topics.sh \
  --bootstrap-server $BBROKERS \
  --command-config config/client-iam.properties \
  --create \
  --topic test_300_1 \
  --partitions 300 \
  --replication-factor 1

# list topics
bin/kafka-topics.sh --list \
  --bootstrap-server $BBROKERS \
  --command-config config/client-iam.properties

# delete topics
for topic in "test_3_3" "test_150_1" "test_300_1"; do
    echo ""
    echo $topic
    bin/kafka-topics.sh --delete \
      --topic $topic \
      --bootstrap-server $BBROKERS \
      --command-config config/client-iam.properties
done;

# get producer perf help
bin/kafka-producer-perf-test.sh --help

# producer perf test
for topic in "test_3_3" "test_150_1" "test_300_1"; do
    echo ""
    echo $topic
    bin/kafka-producer-perf-test.sh \
      --topic $topic \
      --num-records 5000000 \
      --record-size 400 \
      --throughput -1 \
      --producer.config config/producer_perf_v1.properties \
      --print-metrics
done;

# get consumer perf help
bin/kafka-consumer-perf-test.sh --help

# consumer perf test
for topic in "test_3_3" "test_150_1" "test_300_1"; do
    echo ""
    echo $topic
    bin/kafka-consumer-perf-test.sh \
      --topic $topic \
      --messages 5000000 \
      --consumer.config config/consumer_perf_v1.properties \
      --bootstrap-server $BBROKERS
done;

```

## References

- <https://docs.aws.amazon.com/msk/latest/developerguide/bestpractices.html>
- <https://docs.confluent.io/platform/current/installation/configuration/producer-configs.html>
- <https://gist.github.com/ueokande/b96eadd798fff852551b80962862bfb3>
- <https://code.amazon.com/packages/AwsSaMskKdsPerformance/trees/mainline>
- <https://medium.com/metrosystemsro/apache-kafka-how-to-test-performance-for-clients-configured-with-ssl-encryption-3356d3a0d52b>
- <https://developers.redhat.com/blog/2020/04/29/consuming-messages-from-closest-replicas-in-apache-kafka-2-4-0-and-amq-streams#using_the_rack_aware_selector_in_amq_streams>
