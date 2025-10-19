# NiFi + Kafka + Flink + MinIO + Trino Data Pipeline

A complete modern data lakehouse pipeline using Apache NiFi, Kafka, Flink, MinIO (S3), and Trino with real public data.

## 📋 Architecture Overview

```
JSONPlaceholder API → NiFi → Kafka → Flink → MinIO (S3) → Trino (Query Engine)
```

**Data Flow:**
1. **NiFi**: Fetches data from public API, transforms, publishes to Kafka
2. **Kafka**: Message queue for streaming data
3. **Flink**: Real-time processing, enrichment, and analytics
4. **MinIO**: S3-compatible object storage (Data Lake)
5. **Trino**: Distributed SQL query engine for analytics

## 🗂️ Project Structure

```
nifi-kafka-flink-project/
├── docker-compose.yml
├── flink-project/
│   ├── pom.xml
│   └── src/main/java/com/example/
│       └── PostAnalyticsJob.java
├── flink-jars/
├── trino/
│   ├── catalog/
│   │   ├── minio.properties
│   │   └── kafka.properties
│   └── config.properties
└── minio-init/
    └── init-buckets.sh
```

---

## 📄 File Contents

### 1. `docker-compose.yml`

```yaml
version: '3.8'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.6.0
    container_name: zookeeper
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
      ZOOKEEPER_SYNC_LIMIT: 2
    ports:
      - "2181:2181"
    networks:
      - pipeline-net
    volumes:
      - zookeeper_data:/var/lib/zookeeper/data
      - zookeeper_logs:/var/lib/zookeeper/log
    healthcheck:
      test: ["CMD", "nc", "-z", "localhost", "2181"]
      interval: 10s
      timeout: 5s
      retries: 5

  kafka:
    image: confluentinc/cp-kafka:7.6.0
    container_name: kafka
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
      KAFKA_LOG_RETENTION_HOURS: 24
      KAFKA_LOG_SEGMENT_BYTES: 1073741824
      KAFKA_LOG_RETENTION_CHECK_INTERVAL_MS: 300000
    depends_on:
      zookeeper:
        condition: service_healthy
    ports:
      - "9092:9092"
    networks:
      - pipeline-net
    volumes:
      - kafka_data:/var/lib/kafka/data
    healthcheck:
      test: ["CMD", "kafka-broker-api-versions", "--bootstrap-server", "localhost:9092"]
      interval: 10s
      timeout: 10s
      retries: 5
      start_period: 30s

  nifi:
    image: apache/nifi:1.25.0
    container_name: nifi
    environment:
      - NIFI_WEB_HTTP_PORT=8080
      - NIFI_WEB_HTTPS_PORT=
      - NIFI_SECURITY_USER_LOGIN_IDENTITY_PROVIDER=single-user-provider
      - SINGLE_USER_CREDENTIALS_USERNAME=admin
      - SINGLE_USER_CREDENTIALS_PASSWORD=ctsBtRBKHRAx69EqUghvvgEvjnaLjFEB
      - NIFI_SENSITIVE_PROPS_KEY=12345678901234567890123456789012
      - NIFI_JVM_HEAP_INIT=1g
      - NIFI_JVM_HEAP_MAX=2g
      - NIFI_WEB_PROXY_HOST=localhost:8080
    ports:
      - "8080:8080"
    volumes:
      - nifi_data:/opt/nifi/nifi-current/conf
      - nifi_flowfile:/opt/nifi/nifi-current/flowfile_repository
      - nifi_content:/opt/nifi/nifi-current/content_repository
      - nifi_provenance:/opt/nifi/nifi-current/provenance_repository
      - nifi_state:/opt/nifi/nifi-current/state
      - nifi_logs:/opt/nifi/nifi-current/logs
    networks:
      - pipeline-net
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/nifi"]
      interval: 30s
      timeout: 10s
      retries: 10
      start_period: 300s

  minio:
    image: minio/minio:RELEASE.2024-10-13T13-34-11Z
    container_name: minio
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - minio_data:/data
    networks:
      - pipeline-net
    healthcheck:
      test: ["CMD", "mc", "ready", "local"]
      interval: 10s
      timeout: 5s
      retries: 5

  minio-init:
    image: minio/mc:RELEASE.2024-10-08T09-37-26Z
    container_name: minio-init
    depends_on:
      minio:
        condition: service_healthy
    entrypoint: >
      /bin/sh -c "
      mc alias set myminio http://minio:9000 minioadmin minioadmin;
      mc mb --ignore-existing myminio/datalake;
      mc mb --ignore-existing myminio/warehouse;
      mc mb --ignore-existing myminio/raw-data;
      mc mb --ignore-existing myminio/processed-data;
      mc anonymous set download myminio/datalake;
      echo 'MinIO buckets created successfully';
      exit 0;
      "
    networks:
      - pipeline-net

  flink-jobmanager:
    image: flink:1.18-scala_2.12-java11
    container_name: flink-jobmanager
    user: "flink:flink"
    command: >
      bash -c "
      mkdir -p /opt/flink/plugins/s3-fs-hadoop &&
      cp /opt/flink/opt/flink-s3-fs-hadoop-*.jar /opt/flink/plugins/s3-fs-hadoop/ &&
      mkdir -p /tmp/flink-checkpoints /tmp/flink-savepoints &&
      /docker-entrypoint.sh jobmanager
      "
    environment:
      - JOB_MANAGER_RPC_ADDRESS=flink-jobmanager
      - |
        FLINK_PROPERTIES=
        jobmanager.rpc.address: flink-jobmanager
        taskmanager.numberOfTaskSlots: 4
        parallelism.default: 2
        state.backend: filesystem
        state.checkpoints.dir: s3a://datalake/flink-checkpoints
        state.savepoints.dir: s3a://datalake/flink-savepoints
        s3.endpoint: http://minio:9000
        s3.access-key: minioadmin
        s3.secret-key: minioadmin
        s3.path.style.access: true
    ports:
      - "8081:8081"
    volumes:
      - ./flink-jars:/opt/flink/usrlib
      - flink_checkpoints:/tmp/flink-checkpoints
      - flink_savepoints:/tmp/flink-savepoints
    networks:
      - pipeline-net
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8081"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  flink-taskmanager:
    image: flink:1.18-scala_2.12-java11
    container_name: flink-taskmanager
    user: "flink:flink"
    command: >
      bash -c "
      mkdir -p /opt/flink/plugins/s3-fs-hadoop &&
      cp /opt/flink/opt/flink-s3-fs-hadoop-*.jar /opt/flink/plugins/s3-fs-hadoop/ &&
      mkdir -p /tmp/flink-checkpoints &&
      /docker-entrypoint.sh taskmanager
      "
    environment:
      - JOB_MANAGER_RPC_ADDRESS=flink-jobmanager
      - |
        FLINK_PROPERTIES=
        jobmanager.rpc.address: flink-jobmanager
        taskmanager.numberOfTaskSlots: 4
        taskmanager.memory.process.size: 1g
        s3.endpoint: http://minio:9000
        s3.access-key: minioadmin
        s3.secret-key: minioadmin
        s3.path.style.access: true
    depends_on:
      flink-jobmanager:
        condition: service_healthy
    volumes:
      - flink_checkpoints:/tmp/flink-checkpoints
    networks:
      - pipeline-net

  trino:
    image: trinodb/trino:435
    container_name: trino
    ports:
      - "8082:8082"
    volumes:
      - ./trino/catalog:/etc/trino/catalog
      - ./trino/config.properties:/etc/trino/config.properties
    networks:
      - pipeline-net
    depends_on:
      minio-init:
        condition: service_completed_successfully
      kafka:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8082/v1/info"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

networks:
  pipeline-net:
    driver: bridge

volumes:
  zookeeper_data:
  zookeeper_logs:
  kafka_data:
  minio_data:
  nifi_data:
  nifi_flowfile:
  nifi_content:
  nifi_provenance:
  nifi_state:
  nifi_logs:
  flink_checkpoints:
  flink_savepoints:
```

### 2. `trino/config.properties`

```properties
coordinator=true
node-scheduler.include-coordinator=true
http-server.http.port=8080
discovery.uri=http://localhost:8080
query.max-memory=1GB
query.max-memory-per-node=512MB
query.max-total-memory-per-node=768MB
discovery-server.enabled=true
```

### 3. `trino/catalog/minio.properties`

```properties
connector.name=hive
hive.metastore=file
hive.metastore.catalog.dir=/etc/trino/catalog
hive.s3.endpoint=http://minio:9000
hive.s3.path-style-access=true
hive.s3.aws-access-key=minioadmin
hive.s3.aws-secret-key=minioadmin
hive.s3.ssl.enabled=false
hive.non-managed-table-writes-enabled=true
hive.allow-drop-table=true
```

### 4. `trino/catalog/kafka.properties`

```properties
connector.name=kafka
kafka.nodes=kafka:9092
kafka.table-names=posts_raw
kafka.hide-internal-columns=false
```

### 5. `flink-project/pom.xml`

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  
  <groupId>com.example</groupId>
  <artifactId>flink-minio-trino</artifactId>
  <version>1.0-SNAPSHOT</version>
  <packaging>jar</packaging>
  
  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <flink.version>1.18.0</flink.version>
    <scala.binary.version>2.12</scala.binary.version>
    <maven.compiler.source>11</maven.compiler.source>
    <maven.compiler.target>11</maven.compiler.target>
    <jackson.version>2.15.2</jackson.version>
    <hadoop.version>3.3.4</hadoop.version>
  </properties>
  
  <dependencies>
    <!-- Flink Streaming -->
    <dependency>
      <groupId>org.apache.flink</groupId>
      <artifactId>flink-streaming-java</artifactId>
      <version>${flink.version}</version>
      <scope>provided</scope>
    </dependency>
    
    <!-- Kafka Connector -->
    <dependency>
      <groupId>org.apache.flink</groupId>
      <artifactId>flink-connector-kafka</artifactId>
      <version>3.0.1-1.18</version>
    </dependency>
    
    <!-- S3/MinIO FileSystem -->
    <dependency>
      <groupId>org.apache.flink</groupId>
      <artifactId>flink-s3-fs-hadoop</artifactId>
      <version>${flink.version}</version>
    </dependency>
    
    <!-- Parquet Format -->
    <dependency>
      <groupId>org.apache.flink</groupId>
      <artifactId>flink-parquet</artifactId>
      <version>${flink.version}</version>
    </dependency>
    
    <!-- Avro Format -->
    <dependency>
      <groupId>org.apache.flink</groupId>
      <artifactId>flink-avro</artifactId>
      <version>${flink.version}</version>
    </dependency>
    
    <!-- JSON Processing -->
    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-databind</artifactId>
      <version>${jackson.version}</version>
    </dependency>
    
    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-core</artifactId>
      <version>${jackson.version}</version>
    </dependency>
    
    <!-- Hadoop AWS -->
    <dependency>
      <groupId>org.apache.hadoop</groupId>
      <artifactId>hadoop-aws</artifactId>
      <version>${hadoop.version}</version>
      <exclusions>
        <exclusion>
          <groupId>com.fasterxml.jackson.core</groupId>
          <artifactId>jackson-annotations</artifactId>
        </exclusion>
      </exclusions>
    </dependency>
    
    <!-- AWS SDK -->
    <dependency>
      <groupId>com.amazonaws</groupId>
      <artifactId>aws-java-sdk-bundle</artifactId>
      <version>1.12.367</version>
    </dependency>
    
    <!-- Logging -->
    <dependency>
      <groupId>org.slf4j</groupId>
      <artifactId>slf4j-api</artifactId>
      <version>1.7.36</version>
      <scope>provided</scope>
    </dependency>
  </dependencies>
  
  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.11.0</version>
        <configuration>
          <source>11</source>
          <target>11</target>
        </configuration>
      </plugin>
      
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-shade-plugin</artifactId>
        <version>3.5.0</version>
        <executions>
          <execution>
            <phase>package</phase>
            <goals>
              <goal>shade</goal>
            </goals>
            <configuration>
              <artifactSet>
                <excludes>
                  <exclude>org.apache.flink:flink-shaded-force-shading</exclude>
                  <exclude>com.google.code.findbugs:jsr305</exclude>
                  <exclude>org.slf4j:*</exclude>
                  <exclude>org.apache.logging.log4j:*</exclude>
                </excludes>
              </artifactSet>
              <filters>
                <filter>
                  <artifact>*:*</artifact>
                  <excludes>
                    <exclude>META-INF/*.SF</exclude>
                    <exclude>META-INF/*.DSA</exclude>
                    <exclude>META-INF/*.RSA</exclude>
                  </excludes>
                </filter>
              </filters>
              <transformers>
                <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                  <mainClass>com.example.PostAnalyticsJob</mainClass>
                </transformer>
                <transformer implementation="org.apache.maven.plugins.shade.resource.ServicesResourceTransformer"/>
                <transformer implementation="org.apache.maven.plugins.shade.resource.ApacheNoticeResourceTransformer"/>
              </transformers>
            </configuration>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>
```

### 6. `flink-project/src/main/java/com/example/PostAnalyticsJob.java`

```java
package com.example;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import org.apache.flink.api.common.eventtime.WatermarkStrategy;
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.api.common.serialization.SimpleStringEncoder;
import org.apache.flink.api.common.serialization.SimpleStringSchema;
import org.apache.flink.api.java.tuple.Tuple3;
import org.apache.flink.configuration.MemorySize;
import org.apache.flink.connector.file.sink.FileSink;
import org.apache.flink.connector.kafka.source.KafkaSource;
import org.apache.flink.connector.kafka.source.enumerator.initializer.OffsetsInitializer;
import org.apache.flink.core.fs.Path;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.sink.filesystem.OutputFileConfig;
import org.apache.flink.streaming.api.functions.sink.filesystem.bucketassigners.DateTimeBucketAssigner;
import org.apache.flink.streaming.api.functions.sink.filesystem.rollingpolicies.DefaultRollingPolicy;
import org.apache.flink.streaming.api.functions.windowing.ProcessWindowFunction;
import org.apache.flink.streaming.api.windowing.assigners.TumblingProcessingTimeWindows;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.streaming.api.windowing.windows.TimeWindow;
import org.apache.flink.util.Collector;

import java.time.Duration;
import java.time.Instant;
import java.time.ZoneId;
import java.time.format.DateTimeFormatter;

/**
 * Flink job to process posts from JSONPlaceholder API
 * Stores enriched data and analytics in MinIO (S3-compatible storage)
 * Data can be queried using Trino
 *
 * IMPORTANT: S3 configuration is done via docker-compose.yml FLINK_PROPERTIES:
 *   - s3.endpoint: http://minio:9000
 *   - s3.access-key: minioadmin
 *   - s3.secret-key: minioadmin
 *   - s3.path.style.access: true
 */
public class PostAnalyticsJob {

    private static final DateTimeFormatter FORMATTER =
            DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")
                    .withZone(ZoneId.systemDefault());

    // MinIO Configuration (only bucket name needed here)
    private static final String S3_BUCKET = "datalake";

    public static void main(String[] args) throws Exception {
        // 1️⃣ Set up streaming execution environment
        final StreamExecutionEnvironment env =
                StreamExecutionEnvironment.getExecutionEnvironment();

        // 2️⃣ S3/MinIO configuration is automatically loaded from:
        //    - docker-compose.yml FLINK_PROPERTIES
        //    - flink-conf.yaml
        // No manual configuration needed in code!

        // 3️⃣ Enable checkpointing every 30 seconds for fault tolerance
        env.enableCheckpointing(30000);

        // 4️⃣ Configure Kafka Source
        KafkaSource<String> source = KafkaSource.<String>builder()
                .setBootstrapServers("kafka:9092")
                .setTopics("posts_raw")
                .setGroupId("flink-posts-minio-group")
                .setStartingOffsets(OffsetsInitializer.earliest())
                .setValueOnlyDeserializer(new SimpleStringSchema())
                .build();

        // 5️⃣ Create data stream from Kafka
        DataStream<String> rawStream = env
                .fromSource(source, WatermarkStrategy.noWatermarks(), "Kafka Posts Source");

        // 6️⃣ Stream 1: Enrich with processing metadata
        DataStream<String> enrichedStream = rawStream
                .map(new PostEnrichmentMapper())
                .name("Enrich Posts");

        // 7️⃣ Sink 1: Write enriched posts to MinIO (raw zone)
        String enrichedPath = String.format("s3a://%s/raw-data/posts/", S3_BUCKET);

        FileSink<String> enrichedSink = FileSink
                .forRowFormat(new Path(enrichedPath), new SimpleStringEncoder<String>("UTF-8"))
                .withBucketAssigner(new DateTimeBucketAssigner<>("yyyy-MM-dd-HH"))
                .withRollingPolicy(
                        DefaultRollingPolicy.builder()
                                .withRolloverInterval(Duration.ofMinutes(5))
                                .withInactivityInterval(Duration.ofMinutes(2))
                                .withMaxPartSize(MemorySize.ofMebiBytes(128))
                                .build()
                )
                .withOutputFileConfig(
                        OutputFileConfig.builder()
                                .withPartPrefix("enriched-posts")
                                .withPartSuffix(".json")
                                .build()
                )
                .build();

        enrichedStream.sinkTo(enrichedSink).name("Write to MinIO Raw");

        // 8️⃣ Stream 2: Extract user activity for aggregation
        DataStream<Tuple3<Integer, Integer, Long>> userActivity = rawStream
                .map(new UserActivityMapper())
                .name("Extract User Activity");

        // 9️⃣ Stream 3: Windowed aggregation (60 second windows)
        DataStream<String> userStats = userActivity
                .keyBy(tuple -> tuple.f0)
                .window(TumblingProcessingTimeWindows.of(Time.seconds(60)))
                .process(new UserStatsWindowFunction())
                .name("Aggregate User Stats");

        // 🔟 Sink 2: Write aggregated stats to MinIO (processed zone)
        String statsPath = String.format("s3a://%s/processed-data/user-stats/", S3_BUCKET);

        FileSink<String> statsSink = FileSink
                .forRowFormat(new Path(statsPath), new SimpleStringEncoder<String>("UTF-8"))
                .withBucketAssigner(new DateTimeBucketAssigner<>("yyyy-MM-dd-HH"))
                .withRollingPolicy(
                        DefaultRollingPolicy.builder()
                                .withRolloverInterval(Duration.ofMinutes(5))
                                .withInactivityInterval(Duration.ofMinutes(2))
                                .withMaxPartSize(MemorySize.ofMebiBytes(64))
                                .build()
                )
                .withOutputFileConfig(
                        OutputFileConfig.builder()
                                .withPartPrefix("user-stats")
                                .withPartSuffix(".json")
                                .build()
                )
                .build();

        userStats.sinkTo(statsSink).name("Write to MinIO Processed");

        // 🔍 Optional: Print to console for debugging (visible in Flink TaskManager logs)
        enrichedStream.print().name("Print Enriched");
        userStats.print().name("Print Stats");

        // 🚀 Execute the Flink job
        env.execute("JSONPlaceholder to MinIO Pipeline");
    }

    /**
     * MapFunction to enrich posts with processing metadata and analytics
     */
    public static class PostEnrichmentMapper implements MapFunction<String, String> {
        private final ObjectMapper mapper = new ObjectMapper();

        @Override
        public String map(String value) throws Exception {
            JsonNode post = mapper.readTree(value);

            long processingTime = System.currentTimeMillis();
            String formattedTime = FORMATTER.format(Instant.ofEpochMilli(processingTime));

            // Create enriched JSON object
            ObjectNode enriched = mapper.createObjectNode();
            enriched.put("processing_ts", processingTime);
            enriched.put("processing_time", formattedTime);
            enriched.put("ingestion_source", "nifi-kafka-flink");
            enriched.put("storage_layer", "minio-s3");

            // Extract original fields from post
            if (post.has("userId")) enriched.put("user_id", post.get("userId").asInt());
            if (post.has("id")) enriched.put("post_id", post.get("id").asInt());
            if (post.has("title")) enriched.put("title", post.get("title").asText());
            if (post.has("body")) enriched.put("body", post.get("body").asText());

            // Add content analytics
            if (post.has("title")) {
                String title = post.get("title").asText();
                enriched.put("title_length", title.length());
                enriched.put("title_word_count", title.split("\\s+").length);
            }

            if (post.has("body")) {
                String body = post.get("body").asText();
                enriched.put("body_length", body.length());
                enriched.put("body_word_count", body.split("\\s+").length);
            }

            return mapper.writeValueAsString(enriched);
        }
    }

    /**
     * MapFunction to extract user activity data for aggregation
     */
    public static class UserActivityMapper implements MapFunction<String, Tuple3<Integer, Integer, Long>> {
        private final ObjectMapper mapper = new ObjectMapper();

        @Override
        public Tuple3<Integer, Integer, Long> map(String value) throws Exception {
            JsonNode post = mapper.readTree(value);
            int userId = post.has("userId") ? post.get("userId").asInt() : 0;
            long timestamp = System.currentTimeMillis();
            // Return: (userId, count=1, timestamp)
            return new Tuple3<>(userId, 1, timestamp);
        }
    }

    /**
     * ProcessWindowFunction to aggregate user statistics per time window
     */
    public static class UserStatsWindowFunction
            extends ProcessWindowFunction<Tuple3<Integer, Integer, Long>, String, Integer, TimeWindow> {

        private final ObjectMapper mapper = new ObjectMapper();

        @Override
        public void process(Integer userId,
                            Context context,
                            Iterable<Tuple3<Integer, Integer, Long>> elements,
                            Collector<String> out) throws Exception {

            // Aggregate metrics
            int postCount = 0;
            long minTs = Long.MAX_VALUE;
            long maxTs = Long.MIN_VALUE;

            for (Tuple3<Integer, Integer, Long> element : elements) {
                postCount += element.f1;
                minTs = Math.min(minTs, element.f2);
                maxTs = Math.max(maxTs, element.f2);
            }

            long windowStart = context.window().getStart();
            long windowEnd = context.window().getEnd();

            // Create aggregated statistics JSON
            ObjectNode stats = mapper.createObjectNode();
            stats.put("user_id", userId);
            stats.put("post_count", postCount);
            stats.put("window_start", FORMATTER.format(Instant.ofEpochMilli(windowStart)));
            stats.put("window_end", FORMATTER.format(Instant.ofEpochMilli(windowEnd)));
            stats.put("window_duration_seconds", (windowEnd - windowStart) / 1000);
            stats.put("first_post_ts", FORMATTER.format(Instant.ofEpochMilli(minTs)));
            stats.put("last_post_ts", FORMATTER.format(Instant.ofEpochMilli(maxTs)));
            stats.put("processing_time", FORMATTER.format(Instant.now()));

            out.collect(mapper.writeValueAsString(stats));
        }
    }
}
```

---

## 🚀 Setup Instructions

### Step 1: Create Project Structure

```bash
# Create project directory
mkdir -p nifi-kafka-flink-project
cd nifi-kafka-flink-project

# Create all subdirectories
mkdir -p flink-project/src/main/java/com/example
mkdir -p flink-jars
mkdir -p trino/catalog

# Create Trino configuration files
cat > trino/config.properties << 'EOF'
coordinator=true
node-scheduler.include-coordinator=true
http-server.http.port=8080
discovery.uri=http://localhost:8080
query.max-memory=1GB
query.max-memory-per-node=512MB
query.max-total-memory-per-node=768MB
discovery-server.enabled=true
EOF

cat > trino/catalog/minio.properties << 'EOF'
connector.name=hive
hive.metastore=file
hive.metastore.catalog.dir=/etc/trino/catalog
hive.s3.endpoint=http://minio:9000
hive.s3.path-style-access=true
hive.s3.aws-access-key=minioadmin
hive.s3.aws-secret-key=minioadmin
hive.s3.ssl.enabled=false
hive.non-managed-table-writes-enabled=true
hive.allow-drop-table=true
EOF

cat > trino/catalog/kafka.properties << 'EOF'
connector.name=kafka
kafka.nodes=kafka:9092
kafka.table-names=posts_raw
kafka.hide-internal-columns=false
EOF

# Create pom.xml and PostAnalyticsJob.java from above
```

### Step 2: Start Docker Services

```bash
# Start all services
docker compose up -d

# Monitor startup logs
docker compose logs -f

# Wait for all services to be healthy (2-3 minutes)
docker compose ps
```

### Step 3: Verify MinIO Buckets

```bash
# Check MinIO buckets were created
docker exec -it minio mc ls myminio

# Expected output:
# [date] datalake/
# [date] warehouse/
# [date] raw-data/
# [date] processed-data/

# Access MinIO Console: http://localhost:9001
# Login: minioadmin / minioadmin
```

### Step 4: Build Flink Job

```bash
# Navigate to Flink project
cd flink-project

# Build with Maven
mvn clean package

# Copy JAR to flink-jars
cp target/flink-minio-trino-1.0-SNAPSHOT.jar ../flink-jars/

# Return to project root
cd ..
```

**Alternative (no local Maven)**:
```bash
docker run --rm -v "$PWD/flink-project":/app -w /app maven:3.9-eclipse-temurin-11 mvn clean package
cp flink-project/target/flink-minio-trino-1.0-SNAPSHOT.jar flink-jars/
```

### Step 5: Submit Flink Job

**Via Flink Web UI** (http://localhost:8081):
1. Click "Submit New Job"
2. Upload `flink-minio-trino-1.0-SNAPSHOT.jar`
3. Click Submit

**Via CLI**:
```bash
dodocker cp flink-jars/flink-minio-trino-1.0-SNAPSHOT.jar flink-jobmanager:/tmp/
docker exec -it flink-jobmanager /opt/flink/bin/flink run /tmp/flink-minio-trino-1.0-SNAPSHOT.jar
```

### Step 6: Configure NiFi Flow

**Access NiFi**: http://localhost:8080/nifi
- Username: `admin`
- Password: `ctsBtRBKHRAx69EqUghvvgEvjnaLjFEB`

**Create the following processors**:

#### 📥 Processor 1: InvokeHTTP
- **Properties**:
  - `HTTP Method`: `GET`
  - `Remote URL`: `https://jsonplaceholder.typicode.com/posts`
  - `Connection Timeout`: `30 sec`
  - `Read Timeout`: `30 sec`
- **Scheduling**:
  - `Run Schedule`: `60 sec` (avoid rate limiting)
- **Settings**:
  - Auto-terminate: `failure`, `retry`, `no retry`

#### 🔀 Processor 2: SplitJson
- Connect from **InvokeHTTP** → `Response` → **SplitJson**
- **Properties**:
  - `JsonPath Expression`: `$.*`
- **Settings**:
  - Auto-terminate: `failure`, `original`

#### 🏷️ Processor 3: EvaluateJsonPath
- Connect from **SplitJson** → `split` → **EvaluateJsonPath**
- **Properties**:
  - `Destination`: `flowfile-attribute`
  - Add custom properties:
    - `post.userId`: `$.userId`
    - `post.id`: `$.id`
    - `post.title`: `$.title`
- **Settings**:
  - Auto-terminate: `failure`, `unmatched`

#### 📤 Processor 4: PublishKafka_2_6
- Connect from **EvaluateJsonPath** → `matched` → **PublishKafka_2_6**
- **Properties**:
  - `Kafka Brokers`: `kafka:9092`
  - `Topic Name`: `posts_raw`
  - `Delivery Guarantee`: `Best Effort`
- **Settings**:
  - Auto-terminate: `failure`, `success`

**Start the flow** by clicking the play button in the Operate panel.

---

## 🔍 Verification & Data Query

### Step 7: Verify Data Flow to MinIO

```bash
# Check if data is arriving in MinIO
docker exec -it minio mc ls myminio/datalake/raw-data/posts/

# List files in a specific date partition
docker exec -it minio mc ls myminio/datalake/raw-data/posts/2024-10-17-14/

# Download a sample file to inspect
docker exec -it minio mc cp \
  myminio/datalake/raw-data/posts/2024-10-17-14/enriched-posts-0-0.json \
  /tmp/sample.json

docker exec -it minio cat /tmp/sample.json
```

### Step 8: Query Data with Trino

**Access Trino CLI**:
```bash
# Connect to Trino
docker exec -it trino trino --server http://localhost:8082

```

**Query Kafka Stream in Real-time**:

```sql
-- Switch to Kafka catalog
USE kafka.default;

-- Query live Kafka topic
SELECT * FROM posts_raw LIMIT 10;

-- Count messages in Kafka
SELECT COUNT(*) FROM posts_raw;
```
**Create Schema and Tables   To be continue .....**:




---

## 📊 Access Points & Dashboards

| Service | URL | Credentials | Purpose |
|---------|-----|-------------|---------|
| **NiFi UI** | http://localhost:8080/nifi | admin / ctsBtRBKHRAx69EqUghvvgEvjnaLjFEB | Data ingestion flow |
| **Flink Dashboard** | http://localhost:8081 | None | Job monitoring |
| **MinIO Console** | http://localhost:9001 | minioadmin / minioadmin | Object storage browser |
| **Trino UI** | http://localhost:8082 | None | Query execution monitor |
| **Kafka** | localhost:9092 | None | Message broker |

---


---

## 🐛 Troubleshooting

### MinIO Connection Issues

```bash
# Test MinIO connectivity
docker exec -it trino curl http://minio:9000/minio/health/live

# Check MinIO logs
docker compose logs minio

# Verify buckets exist
docker exec -it minio mc ls myminio/
```

### Trino Query Failures

```bash
# Check Trino logs
docker compose logs trino

# Test Trino connectivity
curl http://localhost:8082/v1/info

# Verify catalog configuration
docker exec -it trino cat /etc/trino/catalog/minio.properties
```

**Common Trino Errors**:

1. **"Table not found"**
   - Solution: Run CREATE TABLE statements again
   - Check external_location path matches MinIO

2. **"Access Denied" S3 errors**
   - Solution: Verify MinIO credentials in minio.properties
   - Check bucket permissions

3. **"No nodes available"**
   - Solution: Wait for Trino to fully start (30 seconds)
   - Check: `docker compose ps trino`

### Flink to MinIO Issues

```bash
# Check Flink can reach MinIO
docker exec -it flink-jobmanager curl http://minio:9000/minio/health/live

# View Flink logs for S3 errors
docker compose logs flink-taskmanager | grep -i s3

# Check if files are being created
docker exec -it minio mc ls --recursive myminio/datalake/

# Verify Flink job is running
curl http://localhost:8081/jobs
```

### Kafka Consumer Lag

```bash
# Check consumer group lag
docker exec -it kafka kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group flink-posts-minio-group
```

---

## 🧹 Cleanup & Maintenance

### Stop Services
```bash
# Stop all services
docker compose down

# Stop and remove volumes (deletes all data)
docker compose down -v
```

### Clear MinIO Data Only
```bash
# Delete data in buckets
docker exec -it minio mc rm --recursive --force myminio/datalake/
docker exec -it minio mc rm --recursive --force myminio/raw-data/
docker exec -it minio mc rm --recursive --force myminio/processed-data/
```

### Reset Kafka Topics
```bash
# Delete and recreate topic
docker exec -it kafka kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --delete \
  --topic posts_raw

# Topic will be auto-created on next message
```

### Restart Flink Job
```bash
# Cancel running job
JOBID=$(curl -s http://localhost:8081/jobs | jq -r '.jobs[0].id')
curl -X PATCH http://localhost:8081/jobs/$JOBID?mode=cancel

# Resubmit
docker cp flink-jars/flink-minio-trino-1.0-SNAPSHOT.jar flink-jobmanager:/tmp/
docker exec -it flink-jobmanager /opt/flink/bin/flink run /tmp/flink-minio-trino-1.0-SNAPSHOT.jar

```

---

## 📈 Performance Optimization

### MinIO Tuning

```bash
# Add to docker-compose.yml under minio environment:
- MINIO_API_REQUESTS_MAX=1000
- MINIO_API_REQUESTS_DEADLINE=10s
```

### Flink Tuning

```yaml
# Add to flink-jobmanager environment:
taskmanager.memory.network.fraction: 0.2
taskmanager.memory.managed.fraction: 0.4
state.backend.incremental: true
```

### Trino Query Optimization

```sql
-- Create partitioned table for better performance
CREATE TABLE minio.lakehouse.enriched_posts_partitioned (
    user_id INTEGER,
    post_id INTEGER,
    title VARCHAR,
    body VARCHAR,
    title_length INTEGER,
    body_length INTEGER,
    processing_ts BIGINT
) WITH (
    format = 'JSON',
    external_location = 's3a://datalake/raw-data/posts/',
    partitioned_by = ARRAY['user_id']
);
```

---

## 🎯 What This Pipeline Achieves

✅ **Real-time Data Ingestion**: NiFi pulls from public API every 60 seconds  
✅ **Stream Processing**: Kafka provides reliable message queue  
✅ **Data Enrichment**: Flink adds analytics and metadata  
✅ **Data Lake Storage**: MinIO stores data in S3-compatible format  
**Next step to continue.....**
✅ **SQL Analytics**: Trino enables SQL queries on object storage  
✅ **Time-based Partitioning**: Data organized by hour for efficient queries  
✅ **Multi-Zone Architecture**: Raw and processed data zones  
✅ **Fault Tolerance**: Checkpointing and recovery mechanisms  

---

## 🚀 Next Steps & Extensions

### Add Iceberg Tables
```sql
-- Use Apache Iceberg for ACID transactions
CREATE TABLE minio.lakehouse.posts_iceberg (
    user_id INTEGER,
    post_id INTEGER,
    title VARCHAR,
    processing_time TIMESTAMP
) WITH (
    format = 'PARQUET',
    location = 's3a://warehouse/posts_iceberg'
);
```


## 📚 Additional Resources

- **NiFi Documentation**: https://nifi.apache.org/docs.html
- **Flink S3 Integration**: https://nightlies.apache.org/flink/flink-docs-stable/docs/deployment/filesystems/s3/
- **Trino Hive Connector**: https://trino.io/docs/current/connector/hive.html
- **MinIO Docs**: https://min.io/docs/minio/linux/index.html

---

