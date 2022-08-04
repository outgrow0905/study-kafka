## Async Producer
사실상 Kafka는 Async Producer를 사용하는게 맞다. (52p 참조)  
아래 예시는 Async Producer의 기본 예시이다. 

~~~java
public class MyProducer {
    public void send() {
        Properties properties = new Properties();
        // required setting
        properties.put("bootstrap.servers", "kafka-lb-11608160-eb90449ba349.kr.lb.naverncp.com:9092");
        properties.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        properties.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

        // optional setting
        properties.put("compression.type", "gzip");
        properties.put("linger.ms", 1);
        Producer<String, String> producer = new KafkaProducer<>(properties);
        try {
            for (int i = 0; i < 2; i++) {
                producer.send(
                        new ProducerRecord<>("my-topic", Integer.toString(i), Integer.toString(i)),
                        new MyProducerCallback());
            }
        } catch (Exception e) {
            System.out.println(e);
        } finally {
            producer.close();
        }
    }

    class MyProducerCallback implements Callback {
        @Override
        public void onCompletion(RecordMetadata metadata, Exception exception) {
            if (null != exception) {
                // fail
                exception.printStackTrace();
            } else {
                // success
                System.out.println("send success. metadata: [" + metadata + "]");
                System.out.println("topic: " + metadata.topic());
                System.out.println("partition: " + metadata.partition());
                System.out.println("offset: " + metadata.offset());
            }
        }
    }
}
~~~