
# Kafka SSL and OAuth Troubleshooting Guide

## Problem Statement
After upgrading Kafka to a newer version, the brokers are down. Several properties and files were changed during the upgrade, and the customer does not have an audit of these changes.

A review of the logs reveals the following error:

```
org.apache.kafka.common.config.ConfigException: Invalid value javax.net.ssl.SSLHandshakeException: Empty client certificate chain for configuration
A client SSLEngine created with the provided settings can't connect to a server SSLEngine created with those settings.
```

## Solution Overview
1. Kafka SSL Setup Guide
2. Change the confluent.metrics.reporter.ssl.keystore.password and confluent.metrics.reporter.ssl.key.password.
3. Update the ssl.keystore.password and ssl.key.password.
4. Modify the listener.name.token.oauthbearer.sasl.jaas.config path in kafka1, kafka2, kafka3 server.properties.
5. Update the confluent.metadata.server.token.key.path in kafka1, kafka2, kafka3 server.properties.
6. Adjust the confluent.metadata.topic.replication.factor from 5 to 1.

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
openssl x509 -req -CA ca-cert -CAkey ca-key -in cert-file -out cert-signed -days 365 -CAcreateserial -passin pass:kafka-broker -extensions SAN -extfile <(printf "[SAN]\nsubjectAltName=DNS:kafka1,DNS:localhost")
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

## Step 2: Update Confluent Metrics Reporter Passwords

**Before Configuration:**
```bash
confluent.metrics.reporter.ssl.keystore.password=kafka-training
confluent.metrics.reporter.ssl.key.password=kafka-key
```

**Changed Configuration:**
```bash
confluent.metrics.reporter.ssl.keystore.password=kafka-broker
confluent.metrics.reporter.ssl.key.password=kafka-broker
```

## Step 3: Update SSL Keystore and Key Passwords

**Before Configuration:**
```bash
ssl.keystore.password=kafka-training
ssl.key.password=kafka-key
```

**Changed Configuration:**
```bash
ssl.keystore.password=kafka-broker
ssl.key.password=kafka-broker
```

## Step 4: Update OAuth SASL JAAS Config Path

**Incorrect Configuration:**
```bash
listener.name.token.oauthbearer.sasl.jaas.config=    org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required    publicKeyPath="/etc/kafka/pub.pem";
```

**Corrected Configuration:**
```bash
listener.name.token.oauthbearer.sasl.jaas.config=    org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required    publicKeyPath="/etc/kafka/public.pem";
```

## Step 5: Update Metadata Server Token Key Path

**Incorrect Configuration:**
```bash
confluent.metadata.server.token.key.path=/etc/kafka/mdsKey.pem
```

**Corrected Configuration:**
```bash
confluent.metadata.server.token.key.path=/etc/kafka/tokenKeypair.pem
```

## Step 6: Adjust Metadata Topic Replication Factor

**Incorrect Configuration:**
```bash
confluent.metadata.topic.replication.factor=5
```

**Corrected Configuration:**
```bash
confluent.metadata.topic.replication.factor=1
```

# Result

![Screenshot from 2024-09-27 11-40-35.png](<images/Screenshot from 2024-09-27 11-40-35.png>)
![Screenshot from 2024-09-27 11-38-29.png](<images/Screenshot from 2024-09-27 11-38-29.png>)
![Screenshot from 2024-09-27 11-39-00.png](<images/Screenshot from 2024-09-27 11-39-00.png>)
![Screenshot from 2024-09-27 11-44-49.png](<images/Screenshot from 2024-09-27 11-44-49.png>)
![Screenshot from 2024-09-27 11-35-31.png](<images/Screenshot from 2024-09-27 11-35-31.png>)
![Screenshot from 2024-09-27 11-40-45.png](<images/Screenshot from 2024-09-27 11-40-45.png>)
![Screenshot from 2024-09-27 11-39-13.png](<images/Screenshot from 2024-09-27 11-39-13.png>)
![Screenshot from 2024-09-27 11-38-47.png](<images/Screenshot from 2024-09-27 11-38-47.png>)
![Screenshot from 2024-09-27 11-39-32.png](<images/Screenshot from 2024-09-27 11-39-32.png>)
![Screenshot from 2024-09-27 11-33-07.png](<images/Screenshot from 2024-09-27 11-33-07.png>)
![Screenshot from 2024-09-27 11-33-16.png](<images/Screenshot from 2024-09-27 11-33-16.png>)
![Screenshot from 2024-09-27 11-42-08.png](<images/Screenshot from 2024-09-27 11-42-08.png>)
