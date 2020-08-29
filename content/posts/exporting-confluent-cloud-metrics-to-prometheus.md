+++
keywords = [
  'kafka',
  'confluent',
  'consumer',
  'lag',
  'prometheus',
  'monitoring',
 ]
categories = [
  'Cloud',
  'Docker'
]
title = "Exporting Confluent Cloud Metrics to Prometheus"
date = "2019-02-22T8:00:00Z"
description = "A prometheus exporter for taking metrics from Confluent Cloud"

+++

At Kafka Summit this year, Confluent announced consumption based billing for their [Kafka Cloud](https://confluent.cloud/) offering, making it the cheapest and easiest way to get a Kafka Cluster. However, due to the Kafka cluster being multi-tenanted it comes with some restrictions, ZooKeeper is not exposed and the `__consumer_offsets` topic is restricted, this means popular tools like [Kafka Manager](https://github.com/yahoo/kafka-manager) and [Prometheus Kafka Consumer Group Exporter](https://github.com/braedon/prometheus-kafka-consumer-group-exporter) won't work.

[kafka_exporter](https://github.com/danielqsj/kafka_exporter) comes as a nice alternative as it uses the Kafka Admin Client to access the metrics. However, due to the authentication process required by [Confluent Cloud](https://confluent.cloud/) it doesn't work as is.

By forking [kafka_exporter](https://github.com/imduffy15/kafka_exporter) and upgrading the Kafka client one can get a successful output.

You can try out my build as follows:

```
$ docker run -p 9308:9308 -it imduffy15/kafka_exporter \
--kafka.server=host.region.gcp.confluent.cloud:9092 \
--sasl.enabled \
--sasl.username=username \
--sasl.password="password" \ 
--sasl.handshake \ 
--tls.insecure-skip-tls-verify \
--tls.enabled
```

and on querying `http://localhost:9308/metrics` all metrics documented [here](https://github.com/imduffy15/kafka_exporter#topics) will be available. Prometheus can scrape this and alert and graph on the data.
