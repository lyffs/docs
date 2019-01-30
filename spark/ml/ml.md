# 机器学习 #

-> EX：

	```
	    public void task() throws Exception {
        System.out.println("begin Data Analy");
        /*
        create  a local StreamingContext
         */
        SparkConf conf = new SparkConf().setMaster("local[2]").setAppName("MsgSrvAnaly");

        JavaStreamingContext jc = new JavaStreamingContext(conf, Durations.seconds(1));

        /*
        create a DStream that will connect to hostname:port
         */
        JavaReceiverInputDStream<String> lines = jc.socketTextStream(this.hostname, this.port);

        JavaDStream<String> words = lines.flatMap(new FlatMapFunction<String, String>() {
            public Iterator<String> call(String s) throws Exception {
                String[] elems = s.split(" ");
                String key = "";
                for (int i = 0; i < elems.length; i++) {
                    key = key + elems[i] + "_";
                }
                return Arrays.asList(key).iterator();
            }
        });

        words.foreachRDD(new VoidFunction<JavaRDD<String>>() {
            public void call(JavaRDD<String> stringJavaRDD) throws Exception {
                stringJavaRDD.foreachPartition(new VoidFunction<Iterator<String>>() {


//                    RandomForestClassificationModel Model = RandomForestClassificationModel.load("myModelPath");
                    PipelineModel Model = PipelineModel.load("myModelPath");

                    public void call(Iterator<String> stringIterator) throws Exception {
                        while (stringIterator.hasNext()) {
                            SparkSession Spark = SparkSession.builder().appName("MsgSrvAnaly").getOrCreate();
                            String content = stringIterator.next();
                            String[] ele = content.split("_");

                            Integer f1 = Integer.valueOf(ele[0]);
                            Integer f2 = Integer.valueOf(ele[1]);
                            Integer f3 = Integer.valueOf(ele[2]);
                            Integer f4 = Integer.valueOf(ele[3]);
                            Integer label = Integer.valueOf(ele[4]);

                            System.out.println("record: " + f1 + " " + f2 + " " + f3 + " " + f4 + " " + label);

                            List<Row> LearnData = Arrays.asList(
                                    RowFactory.create(f1, f2, f3, f4, label)
                            );

                            StructType schema = new StructType(new StructField[]{
                                    new StructField("f1", DataTypes.IntegerType, false, Metadata.empty()),
                                    new StructField("f2", DataTypes.IntegerType, false, Metadata.empty()),
                                    new StructField("f3", DataTypes.IntegerType, false, Metadata.empty()),
                                    new StructField("f4", DataTypes.IntegerType, false, Metadata.empty()),
                                    new StructField("label", DataTypes.IntegerType, false, Metadata.empty())
                            });

                            Dataset<Row> testData = Spark.createDataFrame(LearnData, schema);


                            Dataset<Row> predictionResultDF = Model.transform(testData);
                            predictionResultDF.select("f1", "f2", "f3", "f4", "label", "predictedLabel").show(20);
                        }
                    }
                });
            }
        });

//        JavaPairDStream<String, Integer> paris = words.mapToPair(new PairFunction<String, String, Integer>() {
//            public Tuple2<String, Integer> call(String s) throws Exception {
//                return new Tuple2<String, Integer>(s, 1);
//            }
//        });
//
//        JavaPairDStream<String, Integer> wordcounts = paris.reduceByKey(new Function2<Integer, Integer, Integer>() {
//            public Integer call(Integer integer, Integer integer2) throws Exception {
//                return integer + integer2;
//            }
//        });
//
//        wordcounts.print();

        jc.start();
        jc.awaitTermination();
    }
	```