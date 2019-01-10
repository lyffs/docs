# KAFKA #

## 版本 ##

*1.查看kafka版本*
find ./libs/ -name \*kafka_\* | head -1 | grep -o '\kafka[^\n]*'
ex:
	->kafka_2.11-1.0.0-test.jar
	scala verson：2.11
	kafka verion: 1.0.0
## TOPIC ##

*1.1创建TOPIC*

bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic sms

*1.2 生产数据*
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic "sms"

## 数据完整性 ##

->1.生产者数据的不丢失
	kafka的ack机制：在给kafka发送数据的时候，每次发送消息都会有一个确认反馈机制，确保消息正常的能够被收到，其中状态有0,1,-1

	如果是同步模式：ack机制能够保证数据的不丢失。BUT随着leader宕机丢失数据
	如果是异步模式：也会考虑ack的状态，除此之外，异步模式有个buffer,通过buffer来控制数据的发送，有2个值来控制，时间阈值和消息的数量阈值。

->2.消费者数据的不丢失
	通过offset commit的保证数据的不丢失，kafka自己记录了每次消费的offfset数据，下次继续消费的时候，会接着上次的offset进行消费。	