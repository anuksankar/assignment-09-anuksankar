## W205 - Project 3 - Understanding User Behavior Project

### Assignment 9 - Define your Pipeline

A mobile app make API calls to web services when user interacts with it.  The API server handles the actual 
business process and logs events to kafka.

In this assignment, we track two events from a mobile game: buy a sword and join guild. 

As we had created a directory ~/w205/flask-with-kafka during class, we can just cd into it.

```
cd ~/w205/flask-with-kafka
```

Setup

Create a docker-compose.yml with the following 

```
---
version: '2'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 32181
      ZOOKEEPER_TICK_TIME: 2000
    expose:
      - "2181"
      - "2888"
      - "32181"
      - "3888"
    extra_hosts:
      - "moby:127.0.0.1"

  kafka:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:32181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    expose:
      - "9092"
      - "29092"
    extra_hosts:
      - "moby:127.0.0.1"

  mids:
    image: midsw205/base:0.1.8
    stdin_open: true
    tty: true
    volumes:
      - ~/w205:/w205
    expose:
      - "5000"
    ports:
      - "5000:5000"
    extra_hosts:
      - "moby:127.0.0.1"
      
```

Then, spin up the cluster

```
docker-compose up -d
```

Create a topic events

```
docker-compose exec kafka kafka-topics --create --topic events --partitions 1 --replication-factor 1 --if-not-exists --zookeeper zookeeper:32181
```

The following is displayed

```
Created topic "events".
```

Create a game_api_py file by using the vi command
 
```
vi game_api.py
```

Flask

Use the python flask library to write our simple API server

```
#!/usr/bin/env python
from flask import Flask
app = Flask(__name__)

@app.route("/")
def default_response():
    return "\nThis is the default response!\n"

@app.route("/purchase_a_sword")
def purchase_sword():
    # business logic to purchase sword
    return "\nSword Purchased!\n"

@app.route("/join_a_guild")
def join_guild():
    # business logic to join a guild
    return "\nJoined a Guild!\n"
    
```

Save the file ~/w205/flask-with-fafka/game_api.py
In the above file:
* the shebang tells Linux that it is a python script that is executed
* app is a variable holding a pointer to an object of class Flask
* route() is a method in class Flash that is used as a "decorator maker"
* route() is passed a string parameter
* default_response() is the function being decorated


run the game_api.py program using 

```
docker-compose exec mids env FLASK_APP=/w205/flask-with-kafka/game_api.py flask run
```
The following is displayed
```
 * Serving Flask app "game_api"
 * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
```

In another terminal window, test it 
```
docker-compose exec mids curl http://localhost:5000/
```
Displays
```
This is the default response!
```
Running
```
docker-compose exec mids curl http://localhost:5000/purchase_a_sword
```
Displays
```
Sword Purchased!
```
Running
```
docker-compose exec mids curl http://localhost:5000/join_a_guild
```

Displays
```
Joined a Guild!
```
Testing the 2 events a few times displayed the following in the flask window
```
127.0.0.1 - - [18/Nov/2018 22:10:14] "GET / HTTP/1.1" 200 -
127.0.0.1 - - [18/Nov/2018 22:10:57] "GET /purchase_a_sword HTTP/1.1" 200 -
127.0.0.1 - - [18/Nov/2018 22:12:37] "GET /purchase_a_sword HTTP/1.1" 200 -
127.0.0.1 - - [18/Nov/2018 22:20:21] "GET /join_a_guild HTTP/1.1" 200 -
127.0.0.1 - - [18/Nov/2018 22:20:26] "GET /join_a_guild HTTP/1.1" 200 -
127.0.0.1 - - [18/Nov/2018 22:20:30] "GET /join_a_guild HTTP/1.1" 200 -
127.0.0.1 - - [18/Nov/2018 22:20:37] "GET /purchase_a_sword HTTP/1.1" 200 -
^[[B127.0.0.1 - - [18/Nov/2018 22:23:04] "GET /purchase_a_sword HTTP/1.1" 200 -

```
Stop flask
```
Kill flask with control-C
```

Generate events from our webapp and add kafka into the mix 

```
#!/usr/bin/env python
from kafka import KafkaProducer
from flask import Flask
app = Flask(__name__)
event_logger = KafkaProducer(bootstrap_servers='kafka:29092')
events_topic = 'events'

@app.route("/")
def default_response():
    event_logger.send(events_topic, 'default'.encode())
    return "\nThis is the default response!\n"

@app.route("/purchase_a_sword")
def purchase_sword():
    # business logic to purchase sword
    # log event to kafka
    event_logger.send(events_topic, 'purchased_sword'.encode())
    return "\nSword Purchased!\n"

@app.route("/join_a_guild")
def join_guild():
    # business logic to join a guild
    # log event to kafka
    event_logger.send(events_topic, 'joined_guild'.encode())
    return "\nJoined a Guild!\n"
    
```
We have added "from kafka import KafkaProducer" and also the "event_logger.send()" which will 
produce the messages to be consumed by kafkacat.

Run it
```
docker-compose exec mids env FLASK_APP=/w205/flask-with-kafka/game_api.py flask run
```
The following is displayed
```
 * Serving Flask app "game_api"
 * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
```
 
Test it and Generate events
```
docker-compose exec mids curl http://localhost:5000/
```
Displays
```
This is the default response!
```

Runnning
```
docker-compose exec mids curl http://localhost:5000/purchase_a_sword
```
Displays
```
Sword Purchased!
```
Running
```
docker-compose exec mids curl http://localhost:5000/join_a_guild
```
Displays
```
Joined a Guild!
```
Checking the kafka window after testing a few times
``
127.0.0.1 - - [18/Nov/2018 23:28:31] "GET / HTTP/1.1" 200 -

127.0.0.1 - - [18/Nov/2018 23:29:40] "GET /purchase_a_sword HTTP/1.1" 200 -

127.0.0.1 - - [18/Nov/2018 23:29:57] "GET /purchase_a_sword HTTP/1.1" 200 -

127.0.0.1 - - [18/Nov/2018 23:30:49] "GET /join_a_guild HTTP/1.1" 200 -

127.0.0.1 - - [18/Nov/2018 23:31:05] "GET /purchase_a_sword HTTP/1.1" 200 -

127.0.0.1 - - [18/Nov/2018 23:31:10] "GET /join_a_guild HTTP/1.1" 200 -

127.0.0.1 - - [18/Nov/2018 23:31:14] "GET /purchase_a_sword HTTP/1.1" 200 -

```

Read from kafka
Use kafkacat to consume events from the events topic
```
docker-compose exec mids bash -c "kafkacat -C -b kafka:29092 -t events -o beginning -e"
```
displays
```
default
purchased_sword
purchased_sword
joined_guild
purchased_sword
joined_guild
purchased_sword
% Reached end of topic events [0] at offset 7: exiting

```

Tear the cluster down

```
docker-compose down
```
displays
```
Stopping flaskwithkafka_kafka_1 ... done
Stopping flaskwithkafka_mids_1 ... done
Stopping flaskwithkafka_zookeeper_1 ... done
Removing flaskwithkafka_kafka_1 ... done
Removing flaskwithkafka_mids_1 ... done
Removing flaskwithkafka_zookeeper_1 ... done
Removing network flaskwithkafka_default
```
