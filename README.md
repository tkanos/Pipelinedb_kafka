# Event Sourcing with kafka and pipelineDb
Have you ever imagine to create an application without database ? Is it an idea that afraid you ? Let’s see that it is, maybe, not a so crazy idea.

So we will do this tutorial without any database, in order to explain you the philosophy of Event Sourcing, once you got the philosophy nothing prevents you to use a database.

## Let’s start.

Event Storming

First we will design our application to be event driven, and we will take the example of a simple bidding service of an auction site. Using the Event Storming technique. (for more information : https://blog.redelastic.com/corporate-arts-crafts-modelling-reactive-systems-with-event-storming-73c6236f5dd7 )

![even storming](https://cdn-images-1.medium.com/max/1600/1*wW4q-TP-tSnGN2C4qx4ZCA.png)

It should be more complex, it ‘s just a for the example. But on the page of the article, we should see the price of the max bid. And for that instead of doing a SELECT current_price FROM auction (because we don’t have database) we will search on the event store for the last event Bid::BidSucceed that occurred.

If you know Kafka, you will tell me that it’s impossible to search for an event without looking in all events since the beginning, and that could be time consuming. So we can’t work with kafka like it was an event store BUT fortunately for us there are pipelineDb.

## Install Kafka with PipelineDb on Docker

We will set up our environment with docker-compose so for that, please clone the repository : https://github.com/Tkanos/Pipelinedb_kafka.

Once it’s done, in order to start the tutorial, please :
```bash
> docker-compose up
```

If you want to be able to access your kafka locally on your computer, you shouldn’t forget to add kafka to your /etc/hosts :
```bash
echo '127.0.0.1    kafka' >> /etc/hosts
```

Now open a new terminal and enter in your container kafka.
```bash
> docker exec -it <container_kafka_name> bash
```
and then we will create our topic :
```bash
> /opt/kafka/bin/kafka-topics.sh --create --zookeeper zookeeper:2181 --replication-factor 1 --partitions 1 --topic bid-succeed-topic
```
On this topic we will send json messages like :
```json
{ "correlation_id":"ABC", "article_id":123, "user_name":"Felipe", "bid_price":10}
```
Now that we have our topic created let’s play with pipelinedb.
On a third terminal, connect to pipelinedb (using a postgreSql client).
```bash
> psql -p 5432 -h localhost -U pipeline
```
To enable pipeline_kafka, on the psql shell run:
```sql
CREATE EXTENSION pipeline_kafka;
```
pipeline_kafka also needs to know about at least one Kafka server to connect to, so let's make it aware of our local server:
```sql
SELECT pipeline_kafka.add_broker('kafka:9092');
```
The PipelineDB analog to a Kafka topic is a stream, and we’ll need to create a stream that maps to a Kafka topic. Since we’re going to be reading JSON records, our stream can be very simple:
```sql
CREATE STREAM bid_succeed_stream (payload json);
```
Now before going further, let’s just create our continuous view to map to the json sent in the kafka topic :
```sql
CREATE CONTINUOUS VIEW bid_succeed AS SELECT arrival_timestamp, payload->>'correlation_id' AS correlation_id, payload->>'article_id' AS article_id, payload->>'user_name' as user_name, payload->>'bid_price' as bid_price FROM bid_succeed_stream;
```
Once our continuous view is created, let’s consume our topic :
```sql
SELECT pipeline_kafka.consume_begin('bid-succeed-topic', 'bid_succeed_stream', format := 'json');
```
Now you can query your continuous view :
```sql
SELECT * FROM bid_succeed;
```
But as we don’t have produce any data, we will not be able to see a line.
So jump to the terminal where you are in the kafka docker and let’s produce a message :
```bash
> /opt/kafka/bin/kafka-console-producer.sh --broker-list kafka:9092 --topic bid-succeed-topic
```
And send :
```json
{ "correlation_id":"ABC", "article_id":123, "user_name":"Felipe", "bid_price":10}
```
Now if you come back to your psql terminal :
```bash
SELECT * FROM bid_succeed;

arrival_timestamp |correlation_id |article_id |user_name | bid_price
------------------+---------------+-----------+----------+----------
###               | ABC           | 123       | Felipe   | 10
```
If you produce one more message :
```json
{ "correlation_id":"DEF", "article_id":123, "user_name":"Max", "bid_price":20}
```
You will be able to have the max bid price for your article page without database :
```bash
SELECT MAX(bid_price) FROM bid_succeed WHERE article_id='123';
max
-----
 20
(1 row)
```
Just to sum up, instead of quering a database to search some data. We have search in a event store composed of kafka and pipelinedb, the last event produce for the article (or we also can do aggregation of events).

So pipelinedb is a powerful tool, that can do a lot much more (views, transformations, aggregations … )(the link of the documentation is below)

## Connect a real app (golang) :
Now if you want to connect a real app to pipelinedb, you can, it’s really simple you just need a basic postgreSql client in your favorite language. Mine is node.js and golang, so let’s do it golang :
```golang
package main
import (
  "database/sql"
  "fmt"
  _ "github.com/lib/pq"
)
func main() {
  db, err := sql.Open("postgres", "dbname='pipeline' user='pipeline' host='localhost' port=5432 sslmode=disable")
  if err != nil { 
    log.Fatal(err)
  }
  value := "123"
rows, err := db.Query(`SELECT MAX(bid_price) FROM bid_succeed WHERE article_id= $1`, value)
  if err != nil {
    fmt.Println(err)
    os.Exit(-1)
  }
  defer rows.Close()
  for rows.Next() {
    var maxbid string
    if err := rows.Scan(&maxbid); err != nil {
      fmt.Println(err)
    }
    fmt.Printf("%s", maxbid)
  }
  if err := rows.Err(); err != nil {
    log.Fatal(err)
  }
}
```

### Let’s clean our terminals :

In the psql terminal, we will close our consumer and then quit :
```bash
SELECT pipeline_kafka.consume_end('bid-succeed-topic', 'bid_succeed_stream');
\q
```
On the terminal connect to the kafka container in docker, just do a CTRL-C (to quit the producer) and then :
```bash
> exit
```
On the terminal with the docker-compose running do a CTRL-C and then :
```bash
> docker-compose down
```

### Links :
https://medium.com/@felipedutratine/event-sourcing-with-kafka-and-pipelinedb-e2be4b298434
https://kafka.apache.org/quickstart
https://www.pipelinedb.com/blog/sql-on-kafka
http://docs.pipelinedb.com/
