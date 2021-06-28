## IEP06 MsgPack

### Introduction to MsgPack
MessagePack, or in short msgpack, allows us to use a key-value storage but with a better m2m ( Maschine-to-Maschine ) handling, this awesome for data transfers as IPC ( Inter-Process-Communication ) or any other internal used data transfer.

#### How does MessagePack work?
MessagePack is like JSON a fast & key-based format, which allows to serialize and deserialize for communication between computers. Unlike as JSON, msgpack uses binary data instead of JSON. JSON's memory footprint / usage is very high compared to msgpack, but both work very similar.

### Performance
A big benefit of using MsgPack as primary m2m ( Maschine-to-Maschine ) communication is that data will get serialized as optimized as its possible for a key-value format. Its very similar to json, so there wont be an difference in its usage, just how data is being stored. With our benchmark, see below, you'll see the performance difference. As you gain a lot of performance, this would be very useful to add to IntelMQ.

#### Benchmark
During our benchmark we extracted data from spamhaus-drop-collector, prepared it with spamhaus-drop-parser and measured with deduplicator-expert. For this measurement we used 460 messages in total.

JSON median size in bytes: 387
MSGPACK median size in bytes: 329
------------------------
Diff: 16.20%


JSON
* Serialize in ns: 39286
* Deserialize in ns: 30713

MSGPACK
* Serialize in ns: 23483
* Deserialize in ns: 12602

DIFF
* Serialize: 50.35%
* Deserialize: 83.62%