## Consumer Log With AutoCommit True
컨슈머의 `enable.auto.commit=true` 설정을 하면 언제 offset commit을 전송할까?  
메시지 batch를 가져오자마자 전송할까? 아니면, 다음번 poll()을 호출하기 전에 전송할까?  
로그를 한줄씩 살펴보자.


#### - `Fetcher`가 `offset=388`부터 메시지 요청준비를 한다.
~~~
18:50:36.880 [pool-1-thread-1] DEBUG org.apache.kafka.clients.consumer.internals.Fetcher - 
[Consumer clientId=consumer-my-group-id-1, groupId=my-group-id] 
Added READ_UNCOMMITTED fetch request for partition my-topic-0 at position FetchPosition{offset=388, offsetEpoch=Optional[0], currentLeader=LeaderAndEpoch{leader=Optional[192.168.100.9:9092 (id: 1 rack: null)], epoch=0}} to node 192.168.100.9:9092 (id: 1 rack: null)
~~~

#### - 아직 브로커와 세션이 맺어지기 전 상태이다. `(sessionId=INVALID, epoch=INITIAL)` 최초의 요청을 준비한다.
~~~
18:50:36.880 [pool-1-thread-1] DEBUG org.apache.kafka.clients.FetchSessionHandler - 
[Consumer clientId=consumer-my-group-id-1, groupId=my-group-id] 
Built full fetch (sessionId=INVALID, epoch=INITIAL) for node 1 with 1 partition(s).
~~~

#### - 브로커로 요청을 실행한다.
~~~
18:50:36.881 [pool-1-thread-1] DEBUG org.apache.kafka.clients.consumer.internals.Fetcher - 
[Consumer clientId=consumer-my-group-id-1, groupId=my-group-id] 
Sending READ_UNCOMMITTED FullFetchRequest(toSend=(my-topic-0), canUseTopicIds=True) to broker 192.168.100.9:9092 (id: 1 rack: null)

18:50:36.882 [pool-1-thread-1] DEBUG org.apache.kafka.clients.NetworkClient - 
[Consumer clientId=consumer-my-group-id-1, groupId=my-group-id] 
Sending FETCH request with header RequestHeader(apiKey=FETCH, apiVersion=13, clientId=consumer-my-group-id-1, correlationId=10) and timeout 30000 to node 1: FetchRequestData(clusterId=null, replicaId=-1, maxWaitMs=500, minBytes=1, maxBytes=52428800, isolationLevel=0, sessionId=0, sessionEpoch=0, topics=[FetchTopic(topic='my-topic', topicId=3IfoSeXMT7a1vfO5wcDjjw, partitions=[FetchPartition(partition=0, currentLeaderEpoch=0, fetchOffset=388, lastFetchedEpoch=-1, logStartOffset=-1, partitionMaxBytes=1048576)])], forgottenTopicsData=[], rackId='')
~~~

#### - 브로커로부터 수신을 받는다. `highWatermark=399`를 보니 399까지 읽을 수 있는 것 같다. 또한, 레코드의 총 사이즈는 219 byte 인것도 알 수 있다.
~~~ 
18:50:36.892 [pool-1-thread-1] DEBUG org.apache.kafka.clients.NetworkClient - 
[Consumer clientId=consumer-my-group-id-1, groupId=my-group-id] 
Received FETCH response from node 1 for request with header RequestHeader(apiKey=FETCH, apiVersion=13, clientId=consumer-my-group-id-1, correlationId=10): FetchResponseData(throttleTimeMs=0, errorCode=0, sessionId=21233317, responses=[FetchableTopicResponse(topic='', topicId=3IfoSeXMT7a1vfO5wcDjjw, partitions=[PartitionData(partitionIndex=0, errorCode=0, highWatermark=399, lastStableOffset=399, logStartOffset=24, divergingEpoch=EpochEndOffset(epoch=-1, endOffset=-1), currentLeader=LeaderIdAndEpoch(leaderId=-1, leaderEpoch=-1), snapshotId=SnapshotId(endOffset=-1, epoch=-1), abortedTransactions=null, preferredReadReplica=-1, records=MemoryRecords(size=219, buffer=java.nio.HeapByteBuffer[pos=0 lim=219 cap=222]))])])
~~~

#### - `session 21233317` 세션값이 생겼다. 앞으로의 추가적인 fetch 요청은 이 아이디를 계속해서 사용한다.
~~~
18:50:36.893 [pool-1-thread-1] DEBUG org.apache.kafka.clients.FetchSessionHandler - 
[Consumer clientId=consumer-my-group-id-1, groupId=my-group-id] 
Node 1 sent a full fetch response that created a new incremental fetch session 21233317 with 1 response partition(s)
~~~

#### `Fetcher` 가 찍은 로그이다. 이전 `NetworkClient`가 찍은 로그와 거의 비슷하다.
~~~
18:50:36.894 [pool-1-thread-1] DEBUG org.apache.kafka.clients.consumer.internals.Fetcher - 
[Consumer clientId=consumer-my-group-id-1, groupId=my-group-id] 
Fetch READ_UNCOMMITTED at offset 388 for partition my-topic-0 returned fetch data PartitionData(partitionIndex=0, errorCode=0, highWatermark=399, lastStableOffset=399, logStartOffset=24, divergingEpoch=EpochEndOffset(epoch=-1, endOffset=-1), currentLeader=LeaderIdAndEpoch(leaderId=-1, leaderEpoch=-1), snapshotId=SnapshotId(endOffset=-1, epoch=-1), abortedTransactions=null, preferredReadReplica=-1, records=MemoryRecords(size=219, buffer=java.nio.HeapByteBuffer[pos=0 lim=219 cap=222]))
~~~


#### - `enable.auto.commit=true` 설정에 따라, `offset=399`를 전송한다. 이후 데이터는 `offset=399`부터 받으면 된다는 의미이다.
~~~
18:50:47.933 [pool-1-thread-1] DEBUG org.apache.kafka.clients.consumer.internals.ConsumerCoordinator - 
[Consumer clientId=consumer-my-group-id-1, groupId=my-group-id] 
Sending asynchronous auto-commit of offsets {my-topic-0=OffsetAndMetadata{offset=399, leaderEpoch=0, metadata=''}}

18:50:47.933 [pool-1-thread-1] DEBUG org.apache.kafka.clients.NetworkClient - 
[Consumer clientId=consumer-my-group-id-1, groupId=my-group-id]
 Sending OFFSET_COMMIT request with header RequestHeader(apiKey=OFFSET_COMMIT, apiVersion=8, clientId=consumer-my-group-id-1, correlationId=15) and timeout 30000 to node 2147483645: OffsetCommitRequestData(groupId='my-group-id', generationId=40, memberId='consumer-my-group-id-1-c750b095-8aaa-497d-9fbd-0b5c40db0c0e', groupInstanceId=null, retentionTimeMs=-1, topics=[OffsetCommitRequestTopic(name='my-topic', partitions=[OffsetCommitRequestPartition(partitionIndex=0, committedOffset=399, committedLeaderEpoch=0, commitTimestamp=-1, committedMetadata='')])])

18:50:47.943 [pool-1-thread-1] DEBUG org.apache.kafka.clients.NetworkClient - 
[Consumer clientId=consumer-my-group-id-1, groupId=my-group-id] 
Received OFFSET_COMMIT response from node 2147483645 for request with header RequestHeader(apiKey=OFFSET_COMMIT, apiVersion=8, clientId=consumer-my-group-id-1, correlationId=15): OffsetCommitResponseData(throttleTimeMs=0, topics=[OffsetCommitResponseTopic(name='my-topic', partitions=[OffsetCommitResponsePartition(partitionIndex=0, errorCode=0)])])

18:50:47.943 [pool-1-thread-1] DEBUG org.apache.kafka.clients.consumer.internals.ConsumerCoordinator - 
[Consumer clientId=consumer-my-group-id-1, groupId=my-group-id] 
Committed offset 399 for partition my-topic-0

18:50:47.943 [pool-1-thread-1] DEBUG org.apache.kafka.clients.consumer.internals.ConsumerCoordinator - 
[Consumer clientId=consumer-my-group-id-1, groupId=my-group-id] 
Completed asynchronous auto-commit of offsets {my-topic-0=OffsetAndMetadata{offset=399, leaderEpoch=0, metadata=''}}
~~~