type:: [[Database]]
language:: [[CPP]]
category:: [[OLAP]], [[SQL]]

- Example Datasets
	- OnTime
		- OnTime 是美国的民航数据从1987年至今持续更新的数据，跨域30余年，是一个被广泛运用的数据集
		- 下载数据
			- ```shell
			  wget --no-check-certificate --continue https://transtats.bts.gov/PREZIP/On_Time_Reporting_Carrier_On_Time_Performance_1987_present_{1987..2021}_{1..12}.zi`p
			  ```
		- 创建表
			- ```sql
			  CREATE TABLE `ontime`
			  (
			      `Year`                            UInt16,
			      `Quarter`                         UInt8,
			      `Month`                           UInt8,
			      `DayofMonth`                      UInt8,
			      `DayOfWeek`                       UInt8,
			      `FlightDate`                      Date,
			      `Reporting_Airline`               String,
			      `DOT_ID_Reporting_Airline`        Int32,
			      `IATA_CODE_Reporting_Airline`     String,
			      `Tail_Number`                     String,
			      `Flight_Number_Reporting_Airline` String,
			      `OriginAirportID`                 Int32,
			      `OriginAirportSeqID`              Int32,
			      `OriginCityMarketID`              Int32,
			      `Origin`                          FixedString(5),
			      `OriginCityName`                  String,
			      `OriginState`                     FixedString(2),
			      `OriginStateFips`                 String,
			      `OriginStateName`                 String,
			      `OriginWac`                       Int32,
			      `DestAirportID`                   Int32,
			      `DestAirportSeqID`                Int32,
			      `DestCityMarketID`                Int32,
			      `Dest`                            FixedString(5),
			      `DestCityName`                    String,
			      `DestState`                       FixedString(2),
			      `DestStateFips`                   String,
			      `DestStateName`                   String,
			      `DestWac`                         Int32,
			      `CRSDepTime`                      Int32,
			      `DepTime`                         Int32,
			      `DepDelay`                        Int32,
			      `DepDelayMinutes`                 Int32,
			      `DepDel15`                        Int32,
			      `DepartureDelayGroups`            String,
			      `DepTimeBlk`                      String,
			      `TaxiOut`                         Int32,
			      `WheelsOff`                       Int32,
			      `WheelsOn`                        Int32,
			      `TaxiIn`                          Int32,
			      `CRSArrTime`                      Int32,
			      `ArrTime`                         Int32,
			      `ArrDelay`                        Int32,
			      `ArrDelayMinutes`                 Int32,
			      `ArrDel15`                        Int32,
			      `ArrivalDelayGroups`              Int32,
			      `ArrTimeBlk`                      String,
			      `Cancelled`                       UInt8,
			      `CancellationCode`                FixedString(1),
			      `Diverted`                        UInt8,
			      `CRSElapsedTime`                  Int32,
			      `ActualElapsedTime`               Int32,
			      `AirTime`                         Nullable(Int32),
			      `Flights`                         Int32,
			      `Distance`                        Int32,
			      `DistanceGroup`                   UInt8,
			      `CarrierDelay`                    Int32,
			      `WeatherDelay`                    Int32,
			      `NASDelay`                        Int32,
			      `SecurityDelay`                   Int32,
			      `LateAircraftDelay`               Int32,
			      `FirstDepTime`                    String,
			      `TotalAddGTime`                   String,
			      `LongestAddGTime`                 String,
			      `DivAirportLandings`              String,
			      `DivReachedDest`                  String,
			      `DivActualElapsedTime`            String,
			      `DivArrDelay`                     String,
			      `DivDistance`                     String,
			      `Div1Airport`                     String,
			      `Div1AirportID`                   Int32,
			      `Div1AirportSeqID`                Int32,
			      `Div1WheelsOn`                    String,
			      `Div1TotalGTime`                  String,
			      `Div1LongestGTime`                String,
			      `Div1WheelsOff`                   String,
			      `Div1TailNum`                     String,
			      `Div2Airport`                     String,
			      `Div2AirportID`                   Int32,
			      `Div2AirportSeqID`                Int32,
			      `Div2WheelsOn`                    String,
			      `Div2TotalGTime`                  String,
			      `Div2LongestGTime`                String,
			      `Div2WheelsOff`                   String,
			      `Div2TailNum`                     String,
			      `Div3Airport`                     String,
			      `Div3AirportID`                   Int32,
			      `Div3AirportSeqID`                Int32,
			      `Div3WheelsOn`                    String,
			      `Div3TotalGTime`                  String,
			      `Div3LongestGTime`                String,
			      `Div3WheelsOff`                   String,
			      `Div3TailNum`                     String,
			      `Div4Airport`                     String,
			      `Div4AirportID`                   Int32,
			      `Div4AirportSeqID`                Int32,
			      `Div4WheelsOn`                    String,
			      `Div4TotalGTime`                  String,
			      `Div4LongestGTime`                String,
			      `Div4WheelsOff`                   String,
			      `Div4TailNum`                     String,
			      `Div5Airport`                     String,
			      `Div5AirportID`                   Int32,
			      `Div5AirportSeqID`                Int32,
			      `Div5WheelsOn`                    String,
			      `Div5TotalGTime`                  String,
			      `Div5LongestGTime`                String,
			      `Div5WheelsOff`                   String,
			      `Div5TailNum`                     String
			  ) ENGINE = MergeTree
			        PARTITION BY Year
			        ORDER BY (IATA_CODE_Reporting_Airline, FlightDate)
			        SETTINGS index_granularity = 8192;
			  ```
		- 加载数据
			- ```shell
			  ls -1 *.zip | xargs -I{} -P $(nproc) bash -c "echo {}; unzip -cq {} '*.csv' | sed 's/\.00//g' | clickhouse-client --input_format_with_names_use_header=0 --query='INSERT INTO ontime FORMAT CSVWithNames'"
			  ```
	- Star Schema Benchmark
		- SSB 是学术界和工业界广泛使用的一个星型模型测试集([来源](https://www.cs.umb.edu/~poneil/StarSchemaB.PDF))，通过这个测试集合可以方便的对比各种OLAP产品的基础性能指标
		- 下载数据生成器
			- ```shell
			  git clone git@github.com:vadimtk/ssb-dbgen.git
			  cd ssb-dbgen
			  make
			  ```
		- 生成数据
			- ```shell
			  ./dbgen -s
			  ```
		- 创建表
			- ```sql
			  CREATE TABLE customer
			  (
			          C_CUSTKEY       UInt32,
			          C_NAME          String,
			          C_ADDRESS       String,
			          C_CITY          LowCardinality(String),
			          C_NATION        LowCardinality(String),
			          C_REGION        LowCardinality(String),
			          C_PHONE         String,
			          C_MKTSEGMENT    LowCardinality(String)
			  )
			  ENGINE = MergeTree ORDER BY (C_CUSTKEY);
			  
			  CREATE TABLE lineorder
			  (
			      LO_ORDERKEY             UInt32,
			      LO_LINENUMBER           UInt8,
			      LO_CUSTKEY              UInt32,
			      LO_PARTKEY              UInt32,
			      LO_SUPPKEY              UInt32,
			      LO_ORDERDATE            Date,
			      LO_ORDERPRIORITY        LowCardinality(String),
			      LO_SHIPPRIORITY         UInt8,
			      LO_QUANTITY             UInt8,
			      LO_EXTENDEDPRICE        UInt32,
			      LO_ORDTOTALPRICE        UInt32,
			      LO_DISCOUNT             UInt8,
			      LO_REVENUE              UInt32,
			      LO_SUPPLYCOST           UInt32,
			      LO_TAX                  UInt8,
			      LO_COMMITDATE           Date,
			      LO_SHIPMODE             LowCardinality(String)
			  )
			  ENGINE = MergeTree PARTITION BY toYear(LO_ORDERDATE) ORDER BY (LO_ORDERDATE, LO_ORDERKEY);
			  
			  CREATE TABLE part
			  (
			          P_PARTKEY       UInt32,
			          P_NAME          String,
			          P_MFGR          LowCardinality(String),
			          P_CATEGORY      LowCardinality(String),
			          P_BRAND         LowCardinality(String),
			          P_COLOR         LowCardinality(String),
			          P_TYPE          LowCardinality(String),
			          P_SIZE          UInt8,
			          P_CONTAINER     LowCardinality(String)
			  )
			  ENGINE = MergeTree ORDER BY P_PARTKEY;
			  
			  CREATE TABLE supplier
			  (
			          S_SUPPKEY       UInt32,
			          S_NAME          String,
			          S_ADDRESS       String,
			          S_CITY          LowCardinality(String),
			          S_NATION        LowCardinality(String),
			          S_REGION        LowCardinality(String),
			          S_PHONE         String
			  )
			  ENGINE = MergeTree ORDER BY S_SUPPKEY;
			  ```
		- 加载数据
			- ```shell
			  $ clickhouse-client --query "INSERT INTO customer FORMAT CSV" < customer.tbl
			  $ clickhouse-client --query "INSERT INTO part FORMAT CSV" < part.tbl
			  $ clickhouse-client --query "INSERT INTO supplier FORMAT CSV" < supplier.tbl
			  $ clickhouse-client --query "INSERT INTO lineorder FORMAT CSV" < lineorder.tbl
			  ```
		- SSB 测试还可以转化为 `flat schema`：
			- ```sql
			  SET max_memory_usage = 20000000000;
			  
			  CREATE TABLE lineorder_flat
			  ENGINE = MergeTree
			  PARTITION BY toYear(LO_ORDERDATE)
			  ORDER BY (LO_ORDERDATE, LO_ORDERKEY) AS
			  SELECT
			      l.LO_ORDERKEY AS LO_ORDERKEY,
			      l.LO_LINENUMBER AS LO_LINENUMBER,
			      l.LO_CUSTKEY AS LO_CUSTKEY,
			      l.LO_PARTKEY AS LO_PARTKEY,
			      l.LO_SUPPKEY AS LO_SUPPKEY,
			      l.LO_ORDERDATE AS LO_ORDERDATE,
			      l.LO_ORDERPRIORITY AS LO_ORDERPRIORITY,
			      l.LO_SHIPPRIORITY AS LO_SHIPPRIORITY,
			      l.LO_QUANTITY AS LO_QUANTITY,
			      l.LO_EXTENDEDPRICE AS LO_EXTENDEDPRICE,
			      l.LO_ORDTOTALPRICE AS LO_ORDTOTALPRICE,
			      l.LO_DISCOUNT AS LO_DISCOUNT,
			      l.LO_REVENUE AS LO_REVENUE,
			      l.LO_SUPPLYCOST AS LO_SUPPLYCOST,
			      l.LO_TAX AS LO_TAX,
			      l.LO_COMMITDATE AS LO_COMMITDATE,
			      l.LO_SHIPMODE AS LO_SHIPMODE,
			      c.C_NAME AS C_NAME,
			      c.C_ADDRESS AS C_ADDRESS,
			      c.C_CITY AS C_CITY,
			      c.C_NATION AS C_NATION,
			      c.C_REGION AS C_REGION,
			      c.C_PHONE AS C_PHONE,
			      c.C_MKTSEGMENT AS C_MKTSEGMENT,
			      s.S_NAME AS S_NAME,
			      s.S_ADDRESS AS S_ADDRESS,
			      s.S_CITY AS S_CITY,
			      s.S_NATION AS S_NATION,
			      s.S_REGION AS S_REGION,
			      s.S_PHONE AS S_PHONE,
			      p.P_NAME AS P_NAME,
			      p.P_MFGR AS P_MFGR,
			      p.P_CATEGORY AS P_CATEGORY,
			      p.P_BRAND AS P_BRAND,
			      p.P_COLOR AS P_COLOR,
			      p.P_TYPE AS P_TYPE,
			      p.P_SIZE AS P_SIZE,
			      p.P_CONTAINER AS P_CONTAINER
			  FROM lineorder AS l
			  INNER JOIN customer AS c ON c.C_CUSTKEY = l.LO_CUSTKEY
			  INNER JOIN supplier AS s ON s.S_SUPPKEY = l.LO_SUPPKEY
			  INNER JOIN part AS p ON p.P_PARTKEY = l.LO_PARTKEY;
			  ```