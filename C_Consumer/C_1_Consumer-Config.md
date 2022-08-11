## Consumer Config
아래는 가장 기본적인 Consumer 설정이다. Consumer 세부설정을 알아보자.

~~~java
public class MyConsumer {
    public void consume() {
      Properties properties = new Properties();
      
      // required setting
      properties.put("bootstrap.servers", "kafka-lb-11608160-eb90449ba349.kr.lb.naverncp.com:9092");
      properties.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
      properties.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");

      // optional setting
      properties.put("group.id", "test-group-id-1");
      
      KafkaConsumer consumer = new KafkaConsumer(properties);
      consumer.subscribe(Collections.singletonList("my-topic"));

      Duration timeout = Duration.ofMillis(100);

      while(true) {
        ConsumerRecords<String, String> records = consumer.poll(timeout);

        for (ConsumerRecord<String, String> record : records) {
          System.out.printf("topic: %s\n partition: %d\n offset: %d\n key: %s value: %s",
              record.topic(), record.partition(), record.offset(), record.key(), record.value());
        }
      }
    }
}
~~~

## Config Properties
- `fetch.min.bytes`: 최소로 fetch 하고자 하는 데이터 크기이다. 크게 설정할 수록 컨슈머와 broker의 부하를 줄일 수 있다. 예상했겠지만, 이는 latency와 맞바꿔야 한다. 디폴트는 `1` 이다.
- `fetch.max.wait.ms`: 최대 대기시간이다. `fetch.min.bytes`와 같이 설정하면 먼저 충족하는 설정으로 수행된다. 디폴트는 `500` 이다.
- `fetch.max.bytes`: 컨슈머가 poll을 수행할 때마다 받아올 수 이는 데이터의 최대크기이다. 컨슈머가 broker로 부터 받는 데이이터 단위는 `batch`이다. 
만약, `batch`의 크기가 이 설정값을 넘는다면, `batch`는 전송하되 넘는 부분은 잘려진다. consume을 계속 수행하기 위함이다. 디폴트는 `52428800` 이다.
- `max.partition.fetch.bytes`: 하나의 파티션 당 fetch할 최대 데이터크기를 의미한다. 디폴트는 `1048576` 이다. 
사용하지 않을 것을 권장한다. 컨슈머는 해당 브로커에서 몇개의 파티션이 존재하는 지 변동성을 알기 어렵기 때문이다.

- `session.timeout.ms`: 이 시간동안 컨슈머가 heartbeat을 보내지 않는다면 비정상상태로 간주한다. 디폴트는 `45000` 이다.
- `heartbeat.interval.ms`: 컨슈머가 브로커로 heartbeat를 보내는 주기이다. 당연히, `session.timeout.ms` 보다는 짧게 설정해야 하겠다. 
디폴트는 `3000` 이다.
- `max.poll.interval.ms`: 컨슈머가 hang 상태인데, 백그라운드에서 hearbeat을 게속 보낸다면 문제가 될 수 있다. poll 을 기준으로 컨슈머의 상태를 파악하는 것도 좋은 방법이다.
디폴트는 `300000` 이다. 

- `request.timeout.ms`: 이 시간동안 브로커로부터 응답이 없다면 클라이언트는 커넥션을 끊고 다시 연결을 시도한다. 디폴트는 `30000` 이다. 
가이드에서는 더 낮추지는 않기를 권장한다. 브로커에 문제가 있다고 가정하고, 연결시도가 많이 들어온다면 브로커에게 점점 부담이 될 것이기 때문이다.

- `auto.offset.reset`: 컨슈머가 committed offset에 대한 정보가 없을 때, 어디서부터 정보를 받아올 지에 대한 설정이다. 
혹은, 가지고 있는 offset의 값이 브로커에서는 이미 삭제(aged out)되어 없을 수도 있다. `earliest, latest` 두 가지로 설정 가능하다. 디폴트는 `latest` 이다.
- `enable.auto.commit`: 자동으로 브로커로 commit을 보낼지에 대한 설정이다. 주기는 `auto.commit.interval.ms` 으로 설정할 수 있다. 디폴트는 `true` 이다.
consume이 끝나면 자동으로 전송한다. `enable.auto.commit=true` [설정의 로그를 자세히 살펴보자.](C_4_Consumer-Log.md)
- `auto.commit.interval.ms`: async로 계속해서 보내기 때문에, 너무 주기를 빠르게 할 필요는 없다. 오히려 브로커에 부하를 줄 수 있다. 디폴트는 `5000` 이다.

- `group.instance.id`: `Static Group Membership` 에서 사용한다(83p 참조). 디폴트는 `null` 이다.
- `offsets.retention.minutes`: 브로커설정이지만 컨슈머와 밀접한 연관이 있는 설정이다. 컨슈머그룹이 유지된 채로 offset commit을 전송하면, 당연히 이 commit은 계속해서 보관된다.
하지만, 컨슈머그룹이 비어지고 난 뒤에는, 이 설정의 시간이 지나면 이 저장정보는 지워진다. 그리고 완전히 새로운 컨슈머그룹이 등록되는 것처럼 작동한다. 디폴트는 7일 이다.
[[참조 KIP-186]](https://cwiki.apache.org/confluence/display/KAFKA/KIP-186%3A+Increase+offsets+retention+default+to+7+days)


