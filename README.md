Kafka Docker Setup - Scenario interview_sre


Overview

This document guides you through resolving the issue of producing  messages from the domestic_orders topic in a Kafka cluster. 
Prerequisites

Before starting, make sure there are no other versions of the Kafka sandbox running. Stop and remove any previously running Docker containers using:

    docker-compose down -v

    This ensures that there are no residual containers or networks interfering with the current setup.

Starting the Kafka Cluster

    Start the Kafka services using Docker:

    Run the following command to start all necessary Kafka services:

        docker-compose up -d

    The -d flag runs the containers in detached mode, meaning they will run in the background.

    Verify the status of services:

    To ensure that all services are up and running properly, check their status:

        docker-compose ps -a

    Wait until all the services show a "healthy" status before proceeding.


Problem Statement

The client is able to produce to the international_orders topic but receives an error while producing and consuming from domestic_orders using the following commands -



    Producer Command:

            kafka-console-producer --bootstrap-server kafka2:19093 --producer.config /opt/client/client.properties --topic domestic_orders

    Consumer Command:

            kafka-console-consumer --bootstrap-server kafka1:19093 --consumer.config /opt/client/client.properties --from-beginning --topic domestic_orders

    
Step-by-Step Solution

    Step 1: Create Your Own Certificate Authority (CA) and Certificates

    We will create a new CA and generate the required certificates to secure the communication.

        Generate a new CA key and certificate:

            Use the following OpenSSL command to create a Certificate Authority (CA) certificate and key:

                openssl req -new -x509 -days 365 -keyout ca-key -out ca-cert -subj "/C=DE/ST=NRW/L=MS/O=juplo/OU=kafka/CN=Root-CA" -passout pass:kafka-broker

            This creates a CA certificate (ca-cert) and key (ca-key) valid for 365 days, secured by the password kafka-broker.

        Create a Truststore for Kafka brokers:

            The Truststore stores the CA certificate, allowing Kafka brokers to trust the CA:

                keytool -keystore kafka.server.truststore.jks -storepass kafka-broker -import -alias ca-root -file ca-cert -noprompt

            This command imports the ca-cert into the kafka.server.truststore.jks file.

        Create a Keystore for Kafka broker:

            Generate a Keystore and private key for the Kafka broker:

                keytool -keystore kafka.server.keystore.jks -storepass kafka-broker -alias kafka1 -validity 365 -keyalg RSA -genkeypair -keypass kafka-broker -dname "CN=kafka1,OU=kafka,O=juplo,L=MS,ST=NRW,C=DE"

            This command generates a private key with the alias kafka1.

        Generate a Certificate Signing Request (CSR):

            Request a certificate for the Kafka broker:

                keytool -alias kafka1 -keystore kafka.server.keystore.jks -certreq -file cert-file -storepass kafka-broker -keypass kafka-broker

        Sign the certificate using the CA:

            Use the CA to sign the certificate request:

                openssl x509 -req -CA ca-cert -CAkey ca-key -in cert-file -out cert-signed -days 365 -CAcreateserial -passin pass:kafka-broker -extensions SAN -extfile <(printf "\n[SAN]\nsubjectAltName=DNS:kafka1,DNS:localhost")

            This command signs the certificate and outputs a signed certificate called cert-signed.

        Import CA certificate into the Keystore:

            Import the CA certificate into the Kafka broker's Keystore: 

                keytool -importcert -keystore kafka.server.keystore.jks -alias ca-root -file ca-cert -storepass kafka-broker -keypass kafka-broker -noprompt

        Import the signed certificate into the Keystore:

            Finally, import the signed certificate into the Keystore:

                keytool -keystore kafka.server.keystore.jks -alias kafka1 -import -file cert-signed -storepass kafka-broker -keypass kafka-broker -noprompt


    Step 2: Modify Docker Compose Configuration

    Add schema-registery and connect in docker-compose.yml file:
        schema-registry:
            image: confluentinc/cp-schema-registry:7.4.0
            hostname: schema-registry
            container_name: schema-registry
            depends_on:
            - kafka1
            - kafka2
            - kafka3
            command: schema-registry-start /etc/schema-registry/schema-registry.properties
            ports:
            - "8081:8081"
            volumes:
            - ./schema-registry:/etc/schema-registry
            deploy:
            resources:
                limits:
                cpus: "1"
                memory: 512M

        connect:
            image: confluentinc/cp-server-connect:7.4.0
            hostname: connect
            container_name: connect
            depends_on:
            - kafka1
            - kafka2
            - kafka3
            - schema-registry
            command: connect-distributed /etc/kafka/connect-distributed.properties
            ports:
            - "8083:8083"
            volumes:
            - ./connect:/etc/kafka
            deploy:
            resources:
                limits:
                cpus: "2"
                memory: 2048M


    Step 3: Change in kafka2 server.properties:
        Earlier:
           listeners=CLIENT://:29092,BROKER://:29093,TOKEN://:29094
        After:
            listeners=CLIENT://:19093,BROKER://:29093,TOKEN://:29094

    Step4: change in producer.properties in kafka2
       Earlier:
        bootstrap.servers=localhost:9092

       After:
        bootstrap.servers=localhost:19093

    Step5: change in consumer.properties in kafka2

       Earlier:
        bootstrap.servers=localhost:9092

       After:
        bootstrap.servers=localhost:19093
        
    Step 6: Producing and Consuming Messages

        After configuring the Kafka cluster and verifying that the services are running, proceed with producing and consuming messages from the domestic_orders topic.

        Terminal 1: Produce Messages

            Open the Kafka client container:

            Start by entering the Kafka client container with the following command:

                docker exec -it kfkclient bash

            Run the producer command:

            In the container, run the following command to produce messages to the domestic_orders topic:

                kafka-console-producer --bootstrap-server kafka2:19093 --producer.config /opt/client/client.properties --topic domestic_orders

            Type messages:

            Enter a few messages (e.g., "Message 1", "Message 2") and press Enter after each one to produce them.

        Terminal 2: Consume Messages

            Open another terminal for Kafka client container:

            In a second terminal, enter the Kafka client container again:

                docker exec -it kfkclient bash

            Run the consumer command:

            Use the following command to consume messages from the domestic_orders topic:

                kafka-console-consumer --bootstrap-server kafka2:19093 --consumer.config /opt/client/client.properties --from-beginning --topic domestic_orders

    Verify message consumption:

    You should see the messages you produced from the first terminal appear in this terminal. This confirms that the production and consumption processes are working correctly.


Result After Troubleshooting

    By following these steps, the client should now be able to successfully produce and consume messages from the domestic_orders topic. The issue with the kafka2 listner


![Screenshot from 2024-10-07 16-48-55.png](<images/Screenshot from 2024-10-07 16-48-55.png>)
![Screenshot from 2024-10-07 16-48-59.png](<images/Screenshot from 2024-10-07 16-48-59.png>)
