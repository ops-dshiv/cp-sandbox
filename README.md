
# Kafka Clickstream Topic Production Issue Resolution

## Problem Statement
The client is unable to produce to the clickstream topic using the command:

```bash
kafka-console-producer --bootstrap-server kafka1:19092 --producer.config /opt/client/client.properties --topic clickstream
```

The client is using SASL/PLAIN over PLAINTEXT with the user bob and `acks=1`.

## Solution Overview
1. Kafka SSL Setup Guide
2. Modify in `client/setup.sh`
3. Modify in `docker-compose.yml`

## Prerequisites
Ensure no other versions of the Kafka sandbox are running. Clean up any previous Docker containers and volumes by running:

```bash
docker-compose down -v
```

## Step 1: Kafka SSL Setup Guide

### 1.1 Create Your Own Certificate Authority (CA)

```bash
openssl req -new -x509 -days 365 -keyout ca-key -out ca-cert -subj "/C=DE/ST=NRW/L=MS/O=juplo/OU=kafka/CN=Root-CA" -passout pass:kafka-broker
```

### 1.2 Create Truststore and Import Root CA

```bash
keytool -keystore kafka.server.truststore.jks -storepass kafka-broker -import -alias ca-root -file ca-cert -noprompt
```

### 1.3 Create Keystore for Kafka Broker

```bash
keytool -keystore kafka.server.keystore.jks -storepass kafka-broker -alias kafka1 -validity 365 -keyalg RSA -genkeypair -keypass kafka-broker -dname "CN=kafka1,OU=kafka,O=juplo,L=MS,ST=NRW,C=DE"
```

### 1.4 Generate Certificate Signing Request (CSR)

```bash
keytool -alias kafka1 -keystore kafka.server.keystore.jks -certreq -file cert-file -storepass kafka-broker -keypass kafka-broker
```

### 1.5 Sign the Certificate Using the CA

```bash
openssl x509 -req -CA ca-cert -CAkey ca-key -in cert-file -out cert-signed -days 365 -CAcreateserial -passin pass:kafka-broker -extensions SAN -extfile <(printf "[SAN]
subjectAltName=DNS:kafka1,DNS:localhost")
```

### 1.6 Import CA Certificate into Keystore

```bash
keytool -importcert -keystore kafka.server.keystore.jks -alias ca-root -file ca-cert -storepass kafka-broker -keypass kafka-broker -noprompt
```

### 1.7 Import Signed Certificate into Keystore

```bash
keytool -keystore kafka.server.keystore.jks -alias kafka1 -import -file cert-signed -storepass kafka-broker -keypass kafka-broker -noprompt
```

Repeat steps 1.3 to 1.7 for other brokers (kafka2, kafka3).

## Step 2: Modify in `client/setup.sh`

**Incorrect:**
```bash
kafka-topics --bootstrap-server kafka1:19092 --command-config /tmp/client.properties --create --topic domestic_orders --if-not-exists --replica-assignment 2

for x in {1..100}; do echo $x; sleep 2; done | kafka-console-producer --bootstrap-server kafka1:19092 --producer.config /tmp/client.properties --topic domestic_orders

sleep infinity
```

**Correct:**
```bash
kafka-topics --bootstrap-server kafka1:19092 --command-config /opt/client/client.properties --create --topic clickstream --if-not-exists --replica-assignment 2

for x in {1..100}; do echo $x; sleep 2; done | kafka-console-producer --bootstrap-server kafka1:19092 --producer.config /opt/client/client.properties --topic clickstream

sleep infinity
```

## Step 3: Modify in `docker-compose.yml`

### 3.1 Incorrect in docker-compose kafka2 service

```yaml
kafka2:
  image: confluentinc/cp-server:7.4.0
  hostname: kafka2
  container_name: kafka2
  depends_on:
    - zookeeper1
  command: kafka-server-start /etc/kafka/server.properties
  storage_opt:
    size: '512M'
  environment:
    EXTRA_ARGS: -javaagent:/usr/share/jmx-exporter/jmx_prometheus_javaagent-0.18.0.jar=9102:/usr/share/jmx-exporter/kafka_broker.yml
  volumes:
    - ./kafka2:/etc/kafka
    - ./jmx-exporter:/usr/share/jmx-exporter
    - kafka2data:/var/lib/kafka/data
```

**Correct in docker-compose.yml kafka2 service:**
Remove this line in kafka2 service volumes:
```yaml
    - kafka2data:/var/lib/kafka/data
```

### 3.2 Incorrect in docker-compose volumes

```yaml
volumes:
  prometheus_data: {}
  grafana_data: {}
  kafka2data:
    driver_opts:
      type: tmpfs
      o: size=3145728
      device: tmpfs
```

**Correct in docker-compose volumes:**
Remove these lines:
```yaml
  kafka2data:
    driver_opts:
      type: tmpfs
      o: size=3145728
      device: tmpfs
```

## Result
![Screenshot from 2024-10-01 15-19-39.png](<images/Screenshot from 2024-10-01 15-19-39.png>)
![Screenshot from 2024-10-01 15-21-59.png](<images/Screenshot from 2024-10-01 15-21-59.png>)
![Screenshot from 2024-10-01 15-20-39.png](<images/Screenshot from 2024-10-01 15-20-39.png>)
![Screenshot from 2024-10-01 15-20-23.png](<images/Screenshot from 2024-10-01 15-20-23.png>)
![Screenshot from 2024-10-01 15-24-24.png](<images/Screenshot from 2024-10-01 15-24-24.png>)
![Screenshot from 2024-10-01 15-20-58.png](<images/Screenshot from 2024-10-01 15-20-58.png>)
![Screenshot from 2024-10-01 15-19-56.png](<images/Screenshot from 2024-10-01 15-19-56.png>)
![Screenshot from 2024-10-01 15-22-37.png](<images/Screenshot from 2024-10-01 15-22-37.png>)
