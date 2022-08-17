# A demo to show how SR can work with different strategeis

- What's the differences between different strategies?
- How to produce different AVRO serialized event types into the same topic?

## Infra environments for demo

1. Start infra from docker-compose
    ```
    git clone https://github.com/shift-alt-del/otel-playground
    cd otel-playground
    docker-compose start ksqldb-cli ksqldb-server broker zookeeper schema-registry
    ```
2. Create topic
    ```
    # TopicName
    kafka-topics --bootstrap-server localhost:9092 --create --topic topicA --replication-factor 1 --partitions 1

    # RecordName
    kafka-topics --bootstrap-server localhost:9092 --create --topic topicB --replication-factor 1 --partitions 1

    # TopicRecordName
    kafka-topics --bootstrap-server localhost:9092 --create --topic topicC --replication-factor 1 --partitions 1
    ```


## Demo: Data produce with different strategies


Prep:
1. Check schemas
   ```
   python3 -c "with open('./schemas/schema1.avro') as f: print(''.join([l.strip() for l in f.readlines()]))"
   ```
2. Data input
   ```
   {"NAME":"xx"}
   ```

Data produce:
1. TopicName
   ```
   kafka-avro-console-producer \
   --bootstrap-server broker:9092 \
   --topic topicA \
   --property schema.registry.url=http://schema-registry:8081 \
   --property value.schema='{"fields": [{"default": null,"name": "NAME","type": "string"}],"name": "schema1","namespace": "my.sr.demo","type": "record"}'
   ```
2. RecordName
   ```
   kafka-avro-console-producer \
   --bootstrap-server broker:9092 \
   --topic topicB \
   --property schema.registry.url=http://schema-registry:8081 \
   --property value.subject.name.strategy=io.confluent.kafka.serializers.subject.RecordNameStrategy \
   --property value.schema='{"fields": [{"default": null,"name": "NAME","type": "string"}],"name": "schema1","namespace": "my.sr.demo","type": "record"}'
   ```
3. TopicRecordName
   ```
   kafka-avro-console-producer \
   --bootstrap-server broker:9092 \
   --topic topicC \
   --property schema.registry.url=http://schema-registry:8081 \
   --property value.subject.name.strategy=io.confluent.kafka.serializers.subject.TopicRecordNameStrategy \
   --property value.schema='{"fields": [{"default": null,"name": "NAME","type": "string"}],"name": "schema1","namespace": "my.sr.demo","type": "record"}'
   ```

Result:
1. AVRO Console consumer, change `--topic` to consume from `topicA`, `topicB` and `topicC`.
   ```
   kafka-avro-console-consumer \
   --bootstrap-server broker:9092 \
   --topic topicA \
   --property schema.registry.url=http://schema-registry:8081 \
   --from-beginning
   ```

## Demo: Multiple event types

Prep:
1. Check new schema
   ```
   python3 -c "with open('./schemas/schema2.avro') as f: print(''.join([l.strip() for l in f.readlines()]))"
   ```
2. Data input
   ```
   {"TYPE":"xx"}
   ```

Data produce:
1. Produce different schema to topicA -> `org.apache.kafka.common.errors.InvalidConfigurationException: Schema being registered is incompatible with an earlier schema for subject "topicA-value"; error code: 409`
   ```
   kafka-avro-console-producer \
   --bootstrap-server broker:9092 \
   --topic topicA \
   --property schema.registry.url=http://schema-registry:8081 \
   --property value.schema='{"fields": [{"default": null,"name": "TYPE","type": "string"}],"name": "schema999","namespace": "my.sr.demo","type": "record"}'
   ```
2. Produce different schema to topicB/topicC
   ```
   kafka-avro-console-producer \
   --bootstrap-server broker:9092 \
   --topic topicB \
   --property schema.registry.url=http://schema-registry:8081 \
   --property value.schema='{"fields": [{"default": null,"name": "TYPE","type": "string"}],"name": "schema999","namespace": "my.sr.demo","type": "record"}'

   kafka-avro-console-producer \
   --bootstrap-server broker:9092 \
   --topic topicC \
   --property schema.registry.url=http://schema-registry:8081 \
   --property value.schema='{"fields": [{"default": null,"name": "TYPE","type": "string"}],"name": "schema999","namespace": "my.sr.demo","type": "record"}'
   ```

Result:
1. AVRO Console consumer, change `--topic` to consume from `topicA`, `topicB` and `topicC`.
   ```
   kafka-avro-console-consumer \
   --bootstrap-server broker:9092 \
   --topic topicA \
   --property schema.registry.url=http://schema-registry:8081 \
   --from-beginning
   ```

## Demo: Schema Evolution

(Working)

1. Check compatibility
   ```
   jq '. | {schema: tojson}' ./schemas/schema1-evo-bad.avro | curl -X POST http://localhost:8081/compatibility/subjects/topicA-value/versions/latest -H    "Content-Type:application/json" -d @-
   ```