# INFLUXDB #

---

## 1.Concepts ##

### 1.1 RETENTION POLICY ###

*1.创建*

CREATE RETENTION POLICY <retention_policy_name> ON <database_name> DURATION <duration> REPLICATION <n> [SHARD DURATION <duration>] [DEFAULT]

EX:
	CREATE RETENTION POLICY 90_days ON sms DURATION 90d REPLICATION 1 DEFAULT

*2.显示*
SHOW RETENTION POLICIES

### 1.2 INFLUXDB Line Protocol Reference ###

<measurement>[,<tag_key>=<tag_value>[,<tag_key>=<tag_value>]] <field_key>=<field_value>[,<field_key>=<field_value>] [<timestamp>]

描述：
	Mesurement和Field是必须的，Tag和TImestamp是可选的。Tag Key和 Tag value 都是string类型。
	Field Key是string类型，Field Value 类型可以是floats,integers,strings or booleans。Timestamp类型 是 Unix nanosecond timestamp

Tips:
	Use the Network Time Protocol (NTP) to synchronize time between hosts. InfluxDB uses a host’s local time in UTC to assign timestamps to data; if hosts’ clocks aren’t synchronized with NTP, the timestamps on the data written to InfluxDB can be inaccurate.

->1.Data Types
	
	整型 （Integer  64-bit） 整型结尾需要跟随i，EX: 1i

### 1.3 schema_exploration ###

-> 1. 显示TAG的KEY值（SHOW TAG KEYS）

	SHOW TAG KEYS [ON <database_name>] [FROM_clause] [WHERE <tag_key> <operator> ['<tag_value>' | <regular_expression>]] [LIMIT_clause] [OFFSET_clause]

	EX:
		SHOW TAG KEYS ON "sms" FROM "sms_log"	
