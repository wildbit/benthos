Multipart Messaging In Benthos
==============================

Benthos natively supports multipart messages, meaning they can be read,
processed, buffered, and written seemlessly. However, some inputs and outputs do
not support multipart and can therefore cause confusion.

Inputs that do not support multipart are easy, as they are simply read as
multipart messages with one part. Outputs, however, are more tricky. By default,
an output that only supports single part messages (such as kafka), which
receives a multipart message (perhaps from ZMQ), will output a unique message
per part. These parts are 'at-least once', as message delivery can only be
guaranteed for the whole batch of message parts.

These defaults are not always ideal, sometimes we might want to take a multipart
message, encode it into a binary blob, and output that blob as a single part.
The message could then decoded further in the pipeline and seemlessly decoded
back into its original multiple part format.

You can do this with the `multi_to_blob` and `blob_to_multi` processors.

## Why

As a quick example, let's consider a platform that has a service `foo`, which
creates and streams messages to a service `bar`. Up until now these services
would connect directly and send multipart messages over ZMQ, looking like this:

```
Foo (ZMQ) => Bar (ZMQ)
```

After some time we decided that we would like to introduce a kafka cluster so
that messages from `foo` can be replayed independently. In order to save on
development time we decided to use benthos initially to test the idea before
committing. The pipeline now looks like this:

```
Foo (ZMQ) => Benthos => Kafka => Benthos => Bar (ZMQ)
```

With this config for the first benthos:

```yaml
input:
  type: zmq4
  zmq4:
    addresses:
    - tcp://localhost:5555
    socket_type: PULL
output:
  type: kafka
  kafka:
    addresses:
    - localhost:9092
    client_id: benthos_kafka_output
    topic: benthos_stream
processors:
- type: bounds_check
```

And this config for the second benthos:

```yaml
input:
  type: kafka
  kafka:
    addresses:
    - localhost:9092
    consumer_group: benthos_consumer_group
    topic: benthos_stream
    partition: 0
output:
  type: zmq4
  zmq4:
    addresses:
    - tcp://*:5556
    bind: true
    socket_type: PUSH
processors:
- type: bounds_check
```

However, our multipart messages will be written as multiple messages into kafka,
and `bar` incorrectly receives those parts as individual single part messages.
To fix this we introduce a `multi_to_blob` processor to the first benthos
instance, which encodes all messages into single part before passing it to the
output:

```yaml
input:
  type: zmq4
  zmq4:
    addresses:
    - tcp://localhost:5555
    socket_type: PULL
output:
  type: kafka
  kafka:
    addresses:
    - localhost:9092
    client_id: benthos_kafka_output
    topic: benthos_stream
processors:
- type: bounds_check
- type: multi_to_blob
```

And we also add a `blob_to_multi` processor to the second instance, which reads
single part messages as encoded blobs and decodes them into multipart messages
before passing it to the output:

```yaml
input:
  type: kafka
  kafka:
    addresses:
    - localhost:9092
    consumer_group: benthos_consumer_group
    topic: benthos_stream
    partition: 0
output:
  type: zmq4
  zmq4:
    addresses:
    - tcp://*:5556
    bind: true
    socket_type: PUSH
processors:
- type: blob_to_multi
- type: bounds_check
```

Which results in multipart messages at both ends.

The only remaining issue would be if we decided to mix inputs or outputs. For
example, we might decide that we'd like the second instance of benthos to read
from both a kafka and a second `foo`. We could change the previous config to
this:

```yaml
input:
  type: fan_in
  fan_in:
    inputs:
    - type: kafka
      kafka:
        addresses:
        - localhost:9092
        consumer_group: benthos_consumer_group
        topic: benthos_stream
        partition: 0
    - type: zmq4
      zmq4:
        addresses:
        - tcp://localhost:5555
        socket_type: PULL
output:
  type: zmq4
  zmq4:
    addresses:
    - tcp://*:5556
    bind: true
    socket_type: PUSH
processors:
- type: blob_to_multi
- type: bounds_check
```

But it would fail, because our `blob_to_multi` processor would try and decode
the multipart messages from ZMQ which would return an error. We can solve this
by specifying processors per input:

```yaml
input:
  type: fan_in
  fan_in:
    inputs:
    - type: kafka
      kafka:
        addresses:
        - localhost:9092
        consumer_group: benthos_consumer_group
        topic: benthos_stream
        partition: 0
      processors:
      - type: blob_to_multi
    - type: zmq4
      zmq4:
        addresses:
        - tcp://localhost:5555
        socket_type: PULL
output:
  type: zmq4
  zmq4:
    addresses:
    - tcp://*:5556
    bind: true
    socket_type: PUSH
processors:
- type: bounds_check
```

So now we read and decode messages from kafka into multipart messages, but
messages from ZMQ which are already multipart are passed straight through.

Solved.