FROM debezium/connect-base:1.2
WORKDIR $KAFKA_HOME/connect
RUN curl -L -O https://downloads.datastax.com/kafka/kafka-connect-cassandra-sink.tar.gz
RUN tar zxf kafka-connect-cassandra-sink.tar.gz
RUN rm kafka-connect-cassandra-sink.tar.gz