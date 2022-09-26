# Local Development Environment

This defines a local development environment using Docker Compose. It includes a single-node version of Apache Spark with Hudi and a single Kafka broker with Zookeeper as a dependency. Jupyter Notebook is installed on the Apache Spark node as well. 

## Setup

You will need to have Docker and Docker Compose installed on your system to run this. Once they are installed, make sure you are in the root folder of this environment and that you can see the `docker-compose.yaml` file. In your terminal, run:
```
docker compose up -d 
``` 
This will launch the environment and then you can start using it. It will also create a `localdevelopment` directory inside the root of this environment folder. This directory is mapped from your local machine to the container. This is done so that you can develop code locally and run it inside the container as needed. **You need to store all of your code inside of `localdevelopment` if you want to run it in the container.**

### Setting Environment Variables

There are several ways you can add environment variables. You can remote into a container's shell using: `docker exec -it [container name] bash`. You can also set them in the `config` file in either of the service folders (`kafka` and `spark`) in this repo. Additionally, you can modify the `docker-compose` file to add environment variables to a specific container using the `environment` property.

## Spark, Jupyter, and Hudi

The jar files needed to read/write to Hudi and AWS are included in the container already. To open a Jupyter Notebook, open the following link in your browser:
```
http://localhost:8088/tree?token=spudi
```
*Note: You can set the token in the `/spark/config` file. `spudi` is the default token.*

Since the Jupyter Kernel we'll be using is not connected to PySpark by default, we have to initialize the session ourselves. Here is a generic template to use to start your Spark sessions with AWS access and Hudi enabled:
``` python
import os
from pyspark.sql import SparkSession
# You can also set your AWS credentials in the `environment` file in this repo
os.environ["AWS_ACCESS_KEY_ID"] = "Your AWS Access Key ID"
os.environ["AWS_SECRET_ACCESS_KEY"] = "Your AWS Secret Access Key"
# Set your app name here
APP_NAME = "my app name"
spark = SparkSession.builder \
    .appName(APP_NAME) \
    .config('spark.jars', '/opt/bitnami/spark/jars/hudi-spark3.2-bundle_2.12-0.12.0.jar') \
    .config('spark.serializer', 'org.apache.spark.serializer.KryoSerializer') \
    .config('spark.sql.catalog.spark_catalog', 'org.apache.spark.sql.hudi.catalog.HoodieCatalog') \
    .config('spark.sql.extensions', 'org.apache.spark.sql.hudi.HoodieSparkSessionExtension') \
    .getOrCreate()
sc = spark.sparkContext
sc.setLogLevel("OFF") # Adjust the logging level here (set to 'OFF' by default due to hudi's verbosity)
``` 
You can simply copy/paste this into the first cell of any notebook that you need to use Pyspark with and just add in your AWS credentials and update the `APP_NAME`.

### Testing Hudi

If you want to test out Hudi to ensure that it is working, you can run the following code snippet to generate and view sample data:
```python
# Generate hudi trips data sample
tableName = "hudi_trips_cow"
basePath = "file:///tmp/hudi_trips_cow"
dataGen = sc._jvm.org.apache.hudi.QuickstartUtils.DataGenerator()
inserts = sc._jvm.org.apache.hudi.QuickstartUtils.convertToStringList(dataGen.generateInserts(10))
df = spark.read.json(spark.sparkContext.parallelize(inserts, 2))

hudi_options = {
    'hoodie.table.name': tableName,
    'hoodie.datasource.write.recordkey.field': 'uuid',
    'hoodie.datasource.write.partitionpath.field': 'partitionpath',
    'hoodie.datasource.write.table.name': tableName,
    'hoodie.datasource.write.operation': 'upsert',
    'hoodie.datasource.write.precombine.field': 'ts',
    'hoodie.upsert.shuffle.parallelism': 2,
    'hoodie.insert.shuffle.parallelism': 2
}

df.write.format("hudi"). \
    options(**hudi_options). \
    mode("overwrite"). \
    save(basePath)

# Read the Hudi data and show as a pandas dataframe
test = spark.read.format("org.apache.hudi").load(basePath)

test.toPandas().head(-1)
```
## Kafka

The Kafka broker is started up with the `docker compose` command, so there is no setup required on your part. You can access the bootstrap server at: 
```
kafka-broker:9092
``` 
### Testing Kafka

To test Kafka, you can create a producer & consumer to prove that it is working as expected. You can use the following code snippet to test it out:

```python
from kafka import KafkaProducer, KafkaConsumer

producer = KafkaProducer(bootstrap_servers=["kafka-broker:9092"])
consumer = KafkaConsumer("test", bootstrap_servers=["kafka-broker:9092"], auto_offset_reset='earliest')

producer.send("test", value="I am a test".encode('utf-8'))

for message in consumer:
    print(message)
    break
```

This code snippet should print the following message:

```
ConsumerRecord(topic='test', partition=0, offset=0, timestamp=1664225075724, timestamp_type=0, key=None, value=b'I am a test', headers=[], checksum=None, serialized_key_size=-1, serialized_value_size=11, serialized_header_size=-1)
```