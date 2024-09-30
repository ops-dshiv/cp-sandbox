
## Problem Statement

The client just upgraded their SSL certificates used for inter broker communication. The cluster was healthy before the certificate updates. After the certificate updates, the client sees the following error in the broker logs:

```
Caused by: java.util.concurrent.CompletionException: org.apache.kafka.common.errors.TopicAuthorizationException: Not authorized to access topics: [_confluent-metadata-auth]
	at java.base/java.util.concurrent.CompletableFuture.encodeRelay(CompletableFuture.java:367)
	at java.base/java.util.concurrent.CompletableFuture.completeRelay(CompletableFuture.java:376)
	at java.base/java.util.concurrent.CompletableFuture$AnyOf.tryFire(CompletableFuture.java:1663)
	at java.base/java.util.concurrent.CompletableFuture.postComplete(CompletableFuture.java:506)
	at java.base/java.util.concurrent.CompletableFuture.completeExceptionally(CompletableFuture.java:2088)
	at io.confluent.security.auth.provider.ConfluentProvider.lambda$null$10(ConfluentProvider.java:543)
```

## Solution Overview

1. Kafka SSL Setup Guide
2. Kafka Broker Ports Configuration in Docker Compose File

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

## Step 2: Kafka Broker Ports Configuration in Docker Compose File

### For kafka1:

Add the following lines under `ports` in the docker-compose configuration for kafka1:

```yaml
    ports:
      - "19092:19092"  
      - "19093:19093"  
      - "19094:19094"  
```

### For kafka2:

Ensure the following lines are added under `ports` in the docker-compose configuration for kafka2:

```yaml
    ports:
      - "29092:29092"  
      - "29093:29093"  
      - "29094:29094"  
```

### For kafka3:

Add the following lines under `ports` in the docker-compose configuration for kafka3:

```yaml
    ports:
      - "39092:39092"  
      - "39093:39093"  
      - "39094:39094"  
```

## Result![Screenshot from 2024-09-30 20-40-30.png](<images/Screenshot from 2024-09-30 20-40-30.png>)
![Screenshot from 2024-09-30 21-24-14.png](<images/Screenshot from 2024-09-30 21-24-14.png>)
![Screenshot from 2024-09-30 20-40-43.png](<images/Screenshot from 2024-09-30 20-40-43.png>)
![Screenshot from 2024-09-30 21-25-18.png](<images/Screenshot from 2024-09-30 21-25-18.png>)
![Screenshot from 2024-09-30 21-22-48.png](<images/Screenshot from 2024-09-30 21-22-48.png>)
![Screenshot from 2024-09-30 21-26-18.png](<images/Screenshot from 2024-09-30 21-26-18.png>)
![Screenshot from 2024-09-30 21-02-56.png](<images/Screenshot from 2024-09-30 21-02-56.png>)
![Screenshot from 2024-09-30 21-03-02.png](<images/Screenshot from 2024-09-30 21-03-02.png>)
![Screenshot from 2024-09-30 20-40-54.png](<images/Screenshot from 2024-09-30 20-40-54.png>)
![Screenshot from 2024-09-30 21-02-39.png](<images/Screenshot from 2024-09-30 21-02-39.png>)
![Screenshot from 2024-09-30 21-03-24.png](<images/Screenshot from 2024-09-30 21-03-24.png>)
![Screenshot from 2024-09-30 21-22-52.png](<images/Screenshot from 2024-09-30 21-22-52.png>)
![Screenshot from 2024-09-30 20-40-23.png](<images/Screenshot from 2024-09-30 20-40-23.png>)
![Screenshot from 2024-09-30 21-03-14.png](<images/Screenshot from 2024-09-30 21-03-14.png>)
![Screenshot from 2024-09-30 21-23-11.png](<images/Screenshot from 2024-09-30 21-23-11.png>)
![Screenshot from 2024-09-30 21-28-21.png](<images/Screenshot from 2024-09-30 21-28-21.png>)
![Screenshot from 2024-09-30 21-23-02.png](<images/Screenshot from 2024-09-30 21-23-02.png>)
![Screenshot from 2024-09-30 21-22-35.png](<images/Screenshot from 2024-09-30 21-22-35.png>)
![Screenshot from 2024-09-30 21-33-26.png](<images/Screenshot from 2024-09-30 21-33-26.png>)
