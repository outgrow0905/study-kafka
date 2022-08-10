## Consumer Group Management (123p 참조)
Kafka 운영자로서 모니터링해야 할 가장 중요한 정보중 하나는    
각 Consumer Group들이 지연없이 메시지를 처리하고있는 지 여부이다.  
프로듀서가 메시지를 발행하는 속도보다 컨슈머의 처리속도가 현저히 느리다면, 서비스는 정상적으로 동작하지 않을 것이다.  
예를 들어, 은행의 입금메시지를 실시간에 가깝게 처리하지 못한다면 큰 장애가 일어날 것이다.  
따라서, 프로듀서의 메시지전송 속도 대비 컨슈머의 지연정도는 잘 모니터링하고 관리해야한다.  

그리고, 이는 AdminClient API를 이용해볼 수 있다.

## ConsumerGroup 정보확인하기
~~~java
public void describeConsumerGroups(String consumerGroupId) {
        try {
            ConsumerGroupDescription consumerGroupDescription =
                    adminClient.describeConsumerGroups(Arrays.asList(consumerGroupId)).describedGroups().get(consumerGroupId).get();

            System.out.println("Description of group " + consumerGroupId + ": " + consumerGroupDescription);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
    }
    
@Test
void describeConsumerGroups() {
    myAdminClient.describeConsumerGroups("my-group-id");
}
~~~

아래는 로그이다.
~~~
Description of group my-group-id: 
(
groupId=my-group-id, 
isSimpleConsumerGroup=false, 
members=
    (memberId=consumer-my-group-id-3-edb3eeaf-7b78-46be-a189-356001b07231, groupInstanceId=null, clientId=consumer-my-group-id-3, host=/172.16.0.10, assignment=(topicPartitions=)),
    (memberId=consumer-my-group-id-1-5f202c56-a1e8-496b-a004-428a980eabd3, groupInstanceId=null, clientId=consumer-my-group-id-1, host=/172.16.0.10, assignment=(topicPartitions=my-topic-0)),
    (memberId=consumer-my-group-id-2-4c63cfee-a1c8-4f22-b3df-c663dbcc7f07, groupInstanceId=null, clientId=consumer-my-group-id-2, host=/172.16.0.10, assignment=(topicPartitions=)), 
partitionAssignor=range, 
state=Stable, 
coordinator=192.168.100.10:9092 (id: 2 rack: null), 
authorizedOperations=null
)
~~~
ConsumerGroup에 대한 다양한 정보가 포함되어있다.  
하지만, 궁금한 정보인 ConsumerGroup의 컨슈머들의 마지막 offset을 보여주지는 않는다.  
다른 API를 사용해보자.


## ConsumerGroup의 last offset 확인하기 (125p 참조)
~~~java
public void listConsumerGroupOffsets(String consumerGroupId) {
    try {
        Map<TopicPartition, OffsetAndMetadata> offsets =
                adminClient.listConsumerGroupOffsets(consumerGroupId)
                        .partitionsToOffsetAndMetadata().get();
        Map<TopicPartition, OffsetSpec> requestLatestOffsets = new HashMap<>();
        for (TopicPartition tp : offsets.keySet()) {
            requestLatestOffsets.put(tp, OffsetSpec.latest());
        }
        Map<TopicPartition, ListOffsetsResult.ListOffsetsResultInfo> latestOffsets =
                adminClient.listOffsets(requestLatestOffsets).all().get();
        for (Map.Entry<TopicPartition, OffsetAndMetadata> e : offsets.entrySet()) {
            String topic = e.getKey().topic();
            int partition = e.getKey().partition();
            long committedOffset = e.getValue().offset();
            long latestOffset = latestOffsets.get(e.getKey()).offset();
            System.out.println("Consumer group " + consumerGroupId
                    + " has committed offset " + committedOffset
                    + " to topic " + topic + " partition " + partition
                    + ". The latest offset in the partition is "
                    + latestOffset + " so consumer group is "
                    + (latestOffset - committedOffset) + " records behind");
        }
    } catch (InterruptedException e) {
        e.printStackTrace();
    } catch (ExecutionException e) {
        e.printStackTrace();
    }
}

@Test
void listConsumerGroupOffsets() {
    myAdminClient.listConsumerGroupOffsets("my-group-id");
}
~~~

~~~
Consumer group my-group-id has committed offset 235 to topic my-topic partition 0. The latest offset in the partition is 235 so consumer group is 0 records behind
~~~
