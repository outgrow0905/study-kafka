## env
~~~
OS: Ubuntu 20.04

server1: 192.168.100.9 
server2: 192.168.100.10
server3: 192.168.100.11
~~~


## kafka 설치하기
#### install Java
~~~
$ sudo apt-get update && sudo apt-get upgrade
$ sudo apt-get install openjdk-11-jdk
$ java -version # 설치확인
$ vim ~/.bashrc
  export JAVA_HOME=$(dirname $(dirname $(readlink -f $(which java))))
  export PATH=$PATH:$JAVA_HOME/bin
$ source ~/.bashrc
$ echo $JAVA_HOME # 환경변수 세팅 확인
~~~

#### Download kafka
~~~
$ wget https://dlcdn.apache.org/kafka/3.2.0/kafka_2.13-3.2.0.tgz
$ tar -xzf kafka_2.13-3.2.0.tgz
~~~

#### ZooKeeper config
~~~
dataDir: ZooKeeper 관련 데이터들이 저장되는 디렉토리이다.
ticktime: 아래 설정할 initLimit, syncLimit 의 체크주기이다. ms 단위이다.
initLimit: follower가 leader와 연결 시 허용되는 시간이다. ticktime과 곱한 값이 된다.
syncLimit: follower가 leader와 sync되지 않는 상태를 허용하는 시간이다. ticktime과 곱한 값이 된다.
server.X: server.X=hostname:peerPort:leaderPort 포멧이다.
          leaderPort는 leader 선출에 사용되는 포트이다.
          X는 반드시 숫자여야 한다. X는 dataDir에 myid에 설정된 숫자와 같아야 한다.

$ vi bin/zookeeper.properties
  # custom setting
  dataDir=/var/lib/zookeeper
  clientPort=2181
  ticktime=2000
  initLimit=20
  syncLimit=5
  server.1=192.168.100.9:2888:3888
  server.2=192.168.100.10:2888:3888
  server.3=192.168.100.11:2888:3888
  
$ mkdir /var/lib/zookeeper
$ touch /var/lib/zookeeper/myid
$ echo 1 >> /var/lib/zookeeper/myid # 서버별로 1, 2, 3을 부여한다.
~~~

#### ZooKeeper Start
~~~
$ bin/zookeeper-server-start.sh -daemon config/zookeeper.properties
~~~
 
이왕 ZooKeeper를 시작헀으니, [tutorial](A_2_ZooKeeper-tutorial.md)l을 해보는 것도 좋을 것 같다.




## Reference
- https://kafka.apache.org/quickstart
- https://zookeeper.apache.org/doc/current/zookeeperStarted.html