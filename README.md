# dispatchd - A Message Broker and Queue Server

## Status

`dispatchd` is in alpha.

It generally works but is not hardened enough for production use. More on what it can and can't do below

## Features

* Basically all (and many optional) amqp 0-9-1 features are supported
* Some rabbitmq extensions are implemented:
  * nack (almost: the same consumer could receive the message again)
  * internal exchanges (flag exists, but currently unused)
  * auto-delete exchanges
  * Rabbit's reinterpretation of basic.qos
* There is a simple admin page that can show basic info about what's
  happening in the server

Notably missing from the features in the amqp spec are:

* support for multiple priority levels
* handling of queue/memory limits being exceeded

## Configuration

There are command line flags for basic configuration:

    -admin-port int
        Port for admin server. Default: 8080
    -amqp-port int
        Port for amqp protocol messages. Default: 5672
    -config-file string
        Directory for the server and message database files. Default: do not read a config file
    -debug-port int
        Port for the golang debug handlers. Default: 6060
    -persist-dir string
        Directory for the server and message database files. Default: /data/dispatchd/

These options can be overridden if `-config-file` is specified. The config file is JSON and will complain loudly if any types don't look right rather than ignoring or working around them.

Right now the only config file exclusive options are for users and passwords. In the future the config file will have tuning parameters as well.

## Running Dispatchd

Dispatchd is currently only packaged as a docker image. You can run it with this command:

    docker run \
      -p=8080:8080 \
      -p=5672:5672 \
      --volume=YOUR_CONFIG_FILE:/etc/dispatchd.json \
      --volume=YOUR_DATA_DIR:/data/dispatchd/ \
      dispatchd/dispatchd

Config file can be left out for the default behaviors. The data volume needs
to be specified so that data is persisted outside of the container.

## Security/Auth

Dispatchd uses SASL PLAIN auth as required by the amqp spec. There is a default user (user: guest, pw: guest) which is available if there is no config file. If there is a config file the user entries look like this:

    {
      "users" : {
        "guest" : {
          "password_bcrypt_base64" : "JDJhJDExJENobGk4dG5rY0RGemJhTjhsV21xR3VNNnFZZ1ZqTzUzQWxtbGtyMHRYN3RkUHMuYjF5SUt5"
        }
      }
    }

Passwords are generated using bcrypt and then base64 encoded.

## Performance compared to RabbitMQ

All perf testing is done with RabbitMQ's Java perf testing tool. Generally using this command line:

    ./runjava.sh com.rabbitmq.examples.PerfTest --exchange perf-test -uri amqp://guest:guest@localhost:5672 --queue some-queue --consumers 4 --producers 2 --qos 100

On a late 2014 i7 mac mini the results were as follows:

    RabbitMQ Send: ~13000 msg/s, consistent
    RabbitMQ Recv: ~10000 msg/s, consistent
    Dispatchd Send: ~18000 msg/s, varying between 15k and 22k
    Dispatchd Recv: ~18000 msg/s, consistent

It is unclear whether this difference in performance would go away if the server had complete feature parity with Rabbit. Based on the feature diff it isn't clear why it would, but Rabbit is highly tuned and extremely performant.

With the `-flag persistent` performance drops a bit:

    RabbitMQ Send: ~9000k msg/s, varying between 6 and 12k
    RabbitMQ Recv: ~7000k msg/s, consistent
    Dispatchd Send: ~13500k msg/s, varying between 11 and 15k
    Dispatchd Recv: ~13000k msg/s, varying between 11 and 15k

The one thing to note about Dispatchd's send (publish) performance here is that it does not have any internal flow control, so it can get backlogged writing messages to disk. It could be that Rabbit is doing a sustainable 9k and Dispatchd would lose way more messages than come in during one coalesce interval.

On the Receieve (deliver) side, Dispatchd reconciles messages which don't need to by persisted (because they have already been delivered/acked) and so there is no performance hit to persistence if all messages are delivered before the next write to disk happens (every 200ms by default).

## Testing and Code Coverage

Dispatchd has a fairly extensive test suite. Almost all of the major functions are tested and test coverage—ignoring generated code—is around 80%

## What's Next? How do I request changes?

Non-trivial changes are tracked through [github issues](https://github.com/ernestrc/dispatchd/issues).