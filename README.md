# Web-Server-Log-Analysis

## Project Overview
The purpose of this assignment is to implement and execute a Hive-based analytical process for analyzing web server logs. You will create and query a Hive table to process a dataset of web server logs in CSV format and extract meaningful insights about website traffic patterns.

## Instructions

### Step 1: Setup Hive Environment
1. **Start Docker and Resource Manager:**
    ```bash
    docker-compose up -d
    ```

2. **Copy the CSV file to the Docker container:**
    ```bash
    docker cp web_server_logs.csv resourcemanager:/
    ```

3. **Access the Resource Manager container:**
    ```bash
    docker exec -it resourcemanager /bin/bash
    ```

4. **Create a directory to hold the CSV file:**
    ```bash
    hdfs dfs -mkdir /input
    ```

5. **Copy the CSV file into the created folder:**
    ```bash
    hdfs dfs -put 
    ```

### Step 2: Load Data into Hive
1. **Open Hue in your browser at port 8888:**
   - Navigate to `http://localhost:8888`

2. **Create a Hive Database and Table:**
    ```sql
    CREATE EXTERNAL TABLE IF NOT EXISTS web_server_logs (
        ip STRING,
        tstamp STRING,
        url STRING,
        status INT,
        user_agent STRING
    )
    ROW FORMAT DELIMITED
    FIELDS TERMINATED BY ','
    STORED AS TEXTFILE
    LOCATION '/user/hive/warehouse/web_logs';
    ```

3. **Load Data into the Hive Table:** when you run this code the csv filw will be moved from the hdfs into the hive server directory: /user/hive/warr4ehouse/web_logs/web_server_logs.csv
    ```sql
    LOAD DATA INPATH '/user/hive/warehouse/web_logs/web_server_logs.csv' 
    INTO TABLE web_server_logs;
    ```

### Step 3: Execute Queries in Hue
Run the following SQL commands in Hue to perform the analysis:

1. **Total Web Requests:**
    ```sql
    SELECT COUNT(*) AS total_requests FROM web_server_logs;
    ```

2. **Analyze Status Codes:**
    ```sql
    SELECT status, COUNT(*) AS count 
    FROM web_server_logs 
    GROUP BY status 
    ORDER BY count DESC;
    ```

3. **Identify Most Visited Pages:**
    ```sql
    SELECT url, COUNT(*) AS visit_count 
    FROM web_server_logs 
    GROUP BY url 
    ORDER BY visit_count DESC 
    LIMIT 3;
    ```

4. **Traffic Source Analysis:**
    ```sql
    SELECT user_agent, COUNT(*) AS count 
    FROM web_server_logs 
    GROUP BY user_agent 
    ORDER BY count DESC;
    ```

5. **Detect Suspicious Activity:**
    ```sql
    SELECT ip, COUNT(*) AS failed_requests 
    FROM web_server_logs 
    WHERE status IN (404, 500) 
    GROUP BY ip 
    HAVING failed_requests > 3;
    ```

6. **Analyze Traffic Trends:**
    ```sql
    SELECT SUBSTR(tstamp, 1, 16) AS minute, COUNT(*) AS request_count 
    FROM web_server_logs 
    GROUP BY SUBSTR(tstamp, 1, 16) 
    ORDER BY minute;
    ```
7. **Partition the table based on Status code**
    ```sql
    CREATE EXTERNAL TABLE IF NOT EXISTS web_server_logs_partitioned (
    ip STRING,
    tstamp STRING,
    url STRING,
    user_agent STRING
    )
    PARTITIONED BY (status INT)
    ROW FORMAT DELIMITED
    FIELDS TERMINATED BY ','
    STORED AS TEXTFILE
    LOCATION '/user/hive/warehouse/web_logs_partitioned';
    ```
8. **Set dynamic partition mode to nonstrict and load the data into partitioned table**
    ```sql
    SET hive.exec.dynamic.partition = true;
    SET hive.exec.dynamic.partition.mode = nonstrict;
    INSERT INTO TABLE web_server_logs_partitioned PARTITION (status)
    SELECT ip, tstamp, url, user_agent, status FROM web_server_logs;
    ```
9. **run the above queries and expot the result into the hdfs**\
    ```sql
    INSERT OVERWRITE DIRECTORY '/user/hive/warehouse/query_1'
    ROW FORMAT DELIMITED
    FIELDS TERMINATED BY ','
    SELECT COUNT(*) AS total_requests FROM web_server_logs_partitioned;

    INSERT OVERWRITE DIRECTORY '/user/hive/warehouse/query_2'
    ROW FORMAT DELIMITED
    FIELDS TERMINATED BY ','
    SELECT status, COUNT(*) AS count 
    FROM web_server_logs_partitioned 
    GROUP BY status 
    ORDER BY count DESC;

    INSERT OVERWRITE DIRECTORY '/user/hive/warehouse/query_3'
    ROW FORMAT DELIMITED
    FIELDS TERMINATED BY ','
    SELECT url, COUNT(*) AS visit_count 
    FROM web_server_logs_partitioned 
    GROUP BY url
    ORDER BY visit_count DESC
    LIMIT 3;

    INSERT OVERWRITE DIRECTORY '/user/hive/warehouse/query_4'
    ROW FORMAT DELIMITED
    FIELDS TERMINATED BY ','
    SELECT user_agent, COUNT(*) AS count 
    FROM web_server_logs_partitioned 
    GROUP BY user_agent 
    ORDER BY count DESC;

    INSERT OVERWRITE DIRECTORY '/user/hive/warehouse/query_5'
    ROW FORMAT DELIMITED
    FIELDS TERMINATED BY ','
    SELECT ip, COUNT(*) AS failed_requests 
    FROM web_server_logs_partitioned 
    WHERE status IN (404, 500) 
    GROUP BY ip 
    HAVING failed_requests > 3;

    INSERT OVERWRITE DIRECTORY '/user/hive/warehouse/query_6'
    ROW FORMAT DELIMITED
    FIELDS TERMINATED BY ','
    SELECT SUBSTR(timestamp, 1, 16) AS minute, COUNT(*) AS request_count 
    FROM web_server_logs_partitioned 
    GROUP BY SUBSTR(timestamp, 1, 16) 
    ORDER BY minute;
    ```

## Challenges Faced
- Document any issues and how they were resolved during the implementation process.

## Expected Input
- ip,timestamp,url,status,user_agent 192.168.1.1,2024-02-01 10:15:00,/home,200,Mozilla/5.0 192.168.1.2,2024-02-01 10:16:00,/products,200,Chrome/90.0 192.168.1  3,2024-02-01 10:17:00,/checkout,404,Safari/13.1 192.168.1.10,2024-02-01 10:18:00,/home,500,Mozilla/5.0 192.168.1.15,2024-02-01 10:19:00,/products,404,Chrome/ 90.0

## Expected Output
- **Total Web Requests:** 100
- **Status Code Analysis:**
  - 200: 80
  - 404: 10
  - 500: 10
- **Most Visited Pages:**
  - /home: 50
  - /products: 30
  - /checkout: 20
- **Traffic Source Analysis:**
  - Mozilla/5.0: 60
  - Chrome/90.0: 30
  - Safari/13.1: 10
- **Suspicious IP Addresses:**
  - 192.168.1.10: 5 failed requests
  - 192.168.1.15: 4 failed requests
- **Traffic Trend Over Time:**
  - 2024-02-01 10:15: 5 requests
  - 2024-02-01 10:16: 7 requests