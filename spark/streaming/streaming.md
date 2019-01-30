# 实时统计 #

-> 数据流

	源数据 -> 日志 ->kafka -> sparkstreaming -> mysql

	EX:

	1.读取Kafka数据流

	```
	Map<String, String> kafkaParams = new HashMap<String, String>();
        kafkaParams.put("metadata.broker.list", ConstantPool.getInstance().getKafkaConnect());
        kafkaParams.put("group.id", group);
        kafkaParams.put("zookeeper.connection.timeout.ms", "100000");
        kafkaParams.put("zookeeper.sync.time.ms", "5000");
        kafkaParams.put("zookeeper.session.timeout.ms", "100000");
        kafkaParams.put("rebalance.backoff.ms", "30000");
        kafkaParams.put("rebalance.max.retries", "5");
        Set<String> topicsSet = new HashSet<String>(Arrays.asList(ConstantPool.getInstance().getTopic().split(",")));
        JavaPairInputDStream<String, String> directKafkaStream = KafkaUtils.createDirectStream(
                jssc,
                String.class,
                String.class,
                kafka.serializer.StringDecoder.class,
                kafka.serializer.StringDecoder.class,
                kafkaParams,
                topicsSet
        );
	```

	2.Map操作

	```
	JavaDStream<Tuple2<String, String>> apigwStream = directKafkaStream.flatMap(new FlatMapFunction<Tuple2<String, String>, Tuple2<String, String>>() {
            public Iterator<Tuple2<String, String>> call(Tuple2<String, String> line) throws Exception {
                String keyString = null;
                long cost = 0;
                int requestSize = 0;
                int responseSize = 0;
                if (line._2() != null) {
                    try {
                        System.out.println(line._2());
                        JSONObject data = new JSONObject(line._2());
                        String host = data.getString("host");
                        int appid = data.getInt("appid");
                        String method = data.getString("smethod").toLowerCase();
                        int sid = data.getInt("sid");
                        String sname = data.getString("sname");
                        String path = sname + "/" + data.getString("spath");
                        int errcode = data.getInt("errcode");
                        if (errcode != 404) {
                            long startTimeStamp = new Double(data.getDouble("starttime") / 1000000).longValue();
                            long endTimeStamp = new Double(data.getDouble("endtime") / 1000000).longValue();
                            requestSize = data.getInt("request_size");
                            responseSize = data.getInt("response_size");
                            cost = endTimeStamp - startTimeStamp;
                            startTimeStamp = timeTransform(startTimeStamp);
                            int costClass = costClassify(cost);
                            keyString = new StringBuffer().append(sid).append('\01').append(sname).append('\01').append(path).append('\01')
                                    .append(appid).append('\01').append(method).append('\01').append(startTimeStamp / 10000 * 10)
                                    .append('\01').append(costClass).append('\01').append(errcode).append('\01').append(host).toString();
                        }
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
                String value = String.valueOf(cost) + "\t" + requestSize + "\t" + responseSize;

                return Arrays.asList(new Tuple2<String, String>(keyString, value)).iterator();
            }
        });
	```

	3.Reduce操作

	```
	JavaPairDStream<String, String> keyMapStream = filterStream.mapToPair(new PairFunction<Tuple2<String, String>, String, String>() {
            public Tuple2<String, String> call(Tuple2<String, String> s) throws Exception {
                return new Tuple2<String, String>(s._1, s._2 + "\t" + 1);
            }
        }).reduceByKey(new Function2<String, String, String>() {
            public String call(String x, String y) throws Exception {
                String[] pairX = x.split("\t");
                String[] pairY = y.split("\t");
                int costTime = Integer.valueOf(pairX[0]) + Integer.valueOf(pairY[0]);
                int requestSize = Integer.valueOf(pairX[1]) + Integer.valueOf(pairY[1]);
                int responseSize = Integer.valueOf(pairX[2]) + Integer.valueOf(pairY[2]);
                int count = Integer.valueOf(pairX[3]) + Integer.valueOf(pairY[3]);
                return costTime + "\t" + requestSize + "\t" + responseSize + "\t" + count;
            }
        });
	```

	4.入库

	```
	keyMapStream.foreachRDD(new VoidFunction<JavaPairRDD<String, String>>() {
            // @Override
            public void call(JavaPairRDD<String, String> rdd) throws Exception {
                if (!rdd.isEmpty()) {
                    rdd.foreachPartition(new VoidFunction<Iterator<Tuple2<String, String>>>() {
                        //@Override
                        public void call(Iterator<Tuple2<String, String>> partitionOfRecords) throws Exception {
                            String sql = "insert into apigw_analy_data(sid, sname, path, app_id, method,sub_time,resp_time, resp_result,count,gw_host,cost_time,request_size,response_size) values(?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)";
                            //     Connection curConn= DriverManager.getConnection("jdbc:mysql://10.63.4.30:3306/analy_userlog?characterEncoding=UTF-8&autoReconnect=true&failOverReadOnly=false&maxReconnects=10","hive","RdrnZT3FBceZ2t");
                            Connection curConn = DBHelper.getInstance().getCurConnection();
                            PreparedStatement ps = curConn.prepareStatement(sql);
                            while (partitionOfRecords.hasNext()) {
                                Tuple2<String, String> pair = partitionOfRecords.next();
                                try {
                                    String keyString = pair._1();
                                    String[] pairTotal = pair._2().split("\t");
                                    String[] result = keyString.split("\01");
                                    ps.setInt(1, Integer.valueOf(result[0]));
                                    ps.setString(2, result[1]);
                                    ps.setString(3, result[2]);
                                    ps.setInt(4, Integer.valueOf(result[3]));
                                    ps.setString(5, result[4]);
                                    ps.setInt(6, Integer.valueOf(result[5]));
                                    ps.setInt(7, Integer.valueOf(result[6]));
                                    ps.setInt(8, Integer.valueOf(result[7]));
                                    ps.setInt(9, Integer.valueOf(pairTotal[3]));
                                    ps.setString(10, result[8]);
                                    ps.setInt(11, Integer.valueOf(pairTotal[0]));
                                    ps.setInt(12, Integer.valueOf(pairTotal[1]));
                                    ps.setInt(13, Integer.valueOf(pairTotal[2]));
                                    ps.addBatch();
                                } catch (Exception e) {
                                    e.printStackTrace();
                                }
                            }
                            System.out.println("SQL add " + ps.toString());
                            ps.executeBatch();
                            ps.close();
                            curConn.close();
                        }
                    });
                }
            }
        });
	```