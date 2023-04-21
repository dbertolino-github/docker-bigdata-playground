[![Gitter chat](https://badges.gitter.im/gitterHQ/gitter.png)](https://gitter.im/big-data-europe/Lobby)

# Introduction to Docker Big Data Playground

This repository is directly forked and inspired from [Big Data Europe repositories](https://github.com/big-data-europe)

Docker Compose containing:
* Setup a standalone [Apache Spark](https://spark.apache.org/) cluster running one Spark Master and multiple Spark workers
* Setup a standalone Hadoop HDFS cluster

## Running Docker containers 

To start the docker big data playground repository:

    docker-compose up

### Example load data into HDFS

Find the container ID of the namenode:

    docker ps |grep namenode

Copy a data file into the container:

    docker cp data/breweries.csv 1df7a57164de:breweries.csv

Log into the container and put the file into HDFS:

    docker exec -it 1df7a57164de bash
    hdfs dfs -mkdir /data
    hdfs dfs -mkdir /data/openbeer
    hdfs dfs -mkdir /data/openbeer/breweries
    hdfs dfs -put breweries.csv /data/openbeer/breweries/breweries.csv

### Example query HDFS from Spark

Go to http://localhost:8080 on your Docker host (laptop). Here you find the spark:// master address like:
  
    Spark Master at spark://5d35a2ea42ef:7077

Find the container ID of teh spark master container, and connect to the spark scala shell:

    docker ps |grep spark
    docker exec -it 453dd19695b0 bash
    spark/bin/spark-shell --master spark://5d35a2ea42ef:7077

Inside the Spark scala shell execute this commands:

    val df = spark.read.csv("hdfs://namenode:9000/data/openbeer/breweries/breweries.csv")
    df.show()

## Expanding Docker Compose
Add the following services to your `docker-compose.yml` to increase spark worker nodes:
```yml
version: '3'
services:
  spark-worker-2:
    image: bde2020/spark-worker:3.3.0-hadoop3.3
    container_name: spark-worker-2
    depends_on:
      - spark-master
    ports:
      - "8082:8081"
    environment:
      - "SPARK_MASTER=spark://spark-master:7077"
```


## Spark Kubernetes deployment
The BDE Spark images can also be used in a Kubernetes enviroment.

To deploy a simple Spark standalone cluster issue

`kubectl apply -f https://raw.githubusercontent.com/big-data-europe/docker-spark/master/k8s-spark-cluster.yaml`

This will setup a Spark standalone cluster with one master and a worker on every available node using the default namespace and resources. The master is reachable in the same namespace at `spark://spark-master:7077`.
It will also setup a headless service so spark clients can be reachable from the workers using hostname `spark-client`.

Then to use `spark-shell` issue

`kubectl run spark-base --rm -it --labels="app=spark-client" --image bde2020/spark-base:3.3.0-hadoop3.3 -- bash ./spark/bin/spark-shell --master spark://spark-master:7077 --conf spark.driver.host=spark-client`

To use `spark-submit` issue for example

`kubectl run spark-base --rm -it --labels="app=spark-client" --image bde2020/spark-base:3.3.0-hadoop3.3 -- bash ./spark/bin/spark-submit --class CLASS_TO_RUN --master spark://spark-master:7077 --deploy-mode client --conf spark.driver.host=spark-client URL_TO_YOUR_APP`

You can use your own image packed with Spark and your application but when deployed it must be reachable from the workers.
One way to achieve this is by creating a headless service for your pod and then use `--conf spark.driver.host=YOUR_HEADLESS_SERVICE` whenever you submit your application.
