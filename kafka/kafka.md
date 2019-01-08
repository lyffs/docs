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