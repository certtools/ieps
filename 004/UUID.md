# DECISION

By the call 25th of May, 2022, we decided to stay with the IDF intelmq data format as close as possible to AIL's format. There, the UUIDs are UUIDv4. So that nails the decision for IntelMQ's IDF. And UUIDv4 is good enough for the purpose of avoiding routing loops of events.

# Original UUID discussion proposal

Dear intelmq-users & developers,

as of IntelMQ IEP04, we're trying to figure out what requirements have to be met for the uuid.
The most important requirement is: The UUID MUST be 100% unique.

In general there are multiple solutions & implementations for uuids.

||UUIDv1|UUIDv3|UUIDv4|UUIDv5|UUIDv6|Sonyflake|
|--|--|--|--|--|--|--|
|Time-based|✅|❌|❌|❌|✅|✅|
|Time-sortable|❌|❌|❌|❌|✅|✅|
|Includes Randomness|✅|❌|✅|❌|✅|✅|
|Size|128bit (hex)|128bit (hex)|128bit (hex)|128bit (hex)|128bit (hex)|unsigned 64bit
|Info|Major change in Python 3.7||||Currently a draft||

# Quick information
## Used tools
- Using [uuid](http://www.ossp.org/pkg/lib/uuid/) package for decoding
- Using python 2.7.16 & python 3.7.3 to generate the uuids

# UUID based upon [RFC4122](https://datatracker.ietf.org/doc/html/rfc4122)
## UUIDv1
aims to be unique by using the host computers mac address & current date/time.

### IMPORTANT
Changed in version 3.7: Universally administered MAC addresses are preferred over locally administered MAC addresses, since the former are guaranteed to be globally unique, while the latter are not. [Source](https://docs.python.org/3/library/uuid.html#uuid.getnode)

**Major problem**: UUIDv1 acts different in different python versions.

### python 2.x example
```bash
~$ python
>>> import uuid
>>> uuid.uuid1()
UUID('85f5d0bc-f6b3-11eb-acf7-f875a4f5f762')
>>> uuid.uuid1()
UUID('872a60b0-f6b3-11eb-acf7-f875a4f5f762')
```

### python 3.x example
```bash
~$ python3
>>> import uuid
>>> uuid.uuid1()
UUID('b8147e5a-f6b7-11eb-acf7-f875a4f5f762')
>>> uuid.uuid1()
UUID('b8147e5b-f6b7-11eb-acf7-f875a4f5f762')
```

### Decode
```bash
~$ uuid -d b8147e5a-f6b7-11eb-acf7-f875a4f5f762
encode: STR:     b8147e5a-f6b7-11eb-acf7-f875a4f5f762
        SIV:     244684359952094534029854043619699586914
decode: variant: DCE 1.1, ISO/IEC 11578:1996
        version: 1 (time and node based)
        content: time:  2021-08-06 13:10:49.273097.0 UTC
                 clock: 11511 (usually random)
                 node:  f8:ff:ff:ff:ff:ff (global unicast) # NOTE: Anonymized the node address ( privacy )
```

### Conclusion
We can see that the first 8 bytes are random & the following 24 bytes are "static"
Important data we could extract is time information, rest is useless for us.

## UUIDv3
uses so called namespaces & a given value (lack of uniqueness by duplicated namespaces & values)

### IMPORTANT
IMHO **NOT** recommended, as there is no useful information inside the package.

**Major problem**: Uniqueness can be affected by same namespaces & values.

### python 2.x example
```bash
export INTELMQ_NAMESPACE_ID=85f5d0bc-f6b3-11eb-acf7-f875a4f5f762
~$ python
>>> import uuid
>>> import os
>>> uuid.uuid3(uuid.UUID(os.environ["INTELMQ_NAMESPACE_ID"].encode("utf-8")), "test_event")
UUID('b25ec7c4-95f7-3003-814d-06705d5b7016')
```

### python 3.x example
```bash
~$ export INTELMQ_NAMESPACE_ID=85f5d0bc-f6b3-11eb-acf7-f875a4f5f762
~$ python
>>> import uuid
>>> import os
>>> uuid.uuid3(uuid.UUID(os.environ["INTELMQ_NAMESPACE_ID"]), "test_event")
UUID('b25ec7c4-95f7-3003-814d-06705d5b7016')
```

### Decode
```
~$ uuid -d b25ec7c4-95f7-3003-814d-06705d5b7016
encode: STR:     b25ec7c4-95f7-3003-814d-06705d5b7016
        SIV:     237094710929060376531257806116483723286
decode: variant: DCE 1.1, ISO/IEC 11578:1996
        version: 3 (name based, MD5)
        content: B2:5E:C7:C4:95:F7:00:03:01:4D:06:70:5D:5B:70:16
                 (not decipherable: MD5 message digest only)
```

### Conclusion
No relevant data inside the uuid, only used as unique identifier.

## UUIDv4
100% unique, not reproduceable.

### python 2.x example
```bash
~$ python
>>> import uuid
>>> uuid.uuid4()
UUID('949419ec-86eb-4899-aa7d-a4f8eae1117d')
>>> uuid.uuid4()
UUID('fb9b094b-b0ed-4890-88c0-27307d3371e7')
```

### python 3.x example
```bash
~$ python3
>>> import uuid
>>> uuid.uuid4()
UUID('7cd42e22-d791-4f0e-a57e-41aa371b95ee')
>>> uuid.uuid4()
UUID('91ac266b-a17a-45aa-8e63-ab021e66cf18')
```

### Decode
```bash
encode: STR:     7cd42e22-d791-4f0e-a57e-41aa371b95ee
        SIV:     165925974162653189851712436253325235694
decode: variant: DCE 1.1, ISO/IEC 11578:1996
        version: 4 (random data based)
        content: 7C:D4:2E:22:D7:91:0F:0E:25:7E:41:AA:37:1B:95:EE
                 (no semantics: random data only)
```

### Conclusion
Totally random data, so its hard to reproduce the given uuid.

## UUIDv5
Generate a UUID based on the SHA-1 hash of a namespace identifier (which is a UUID) and a name (which is a string). [Source](https://docs.python.org/3/library/uuid.html#uuid.uuid5)

### python 2.x example
```bash
~$ export INTELMQ_NAMESPACE_ID=85f5d0bc-f6b3-11eb-acf7-f875a4f5f762
~$ python
>>> import uuid
>>> import os
>>> uuid.uuid5(uuid.UUID(os.environ["INTELMQ_NAMESPACE_ID"]), "test_event")
UUID('d7ced470-31e7-54ed-953f-efc133f40ad0')
>>> uuid.uuid5(uuid.UUID(os.environ["INTELMQ_NAMESPACE_ID"]), "test_event")
UUID('d7ced470-31e7-54ed-953f-efc133f40ad0')
```

### python 3.x example
```bash
~$ export INTELMQ_NAMESPACE_ID=85f5d0bc-f6b3-11eb-acf7-f875a4f5f762
~$ python3
>>> import uuid
>>> import os
>>> uuid.uuid5(uuid.UUID(os.environ["INTELMQ_NAMESPACE_ID"]), "test_event")
UUID('d7ced470-31e7-54ed-953f-efc133f40ad0')
>>> uuid.uuid5(uuid.UUID(os.environ["INTELMQ_NAMESPACE_ID"]), "test_event")
UUID('d7ced470-31e7-54ed-953f-efc133f40ad0')
```

### Decode
```bash
encode: STR:     d7ced470-31e7-54ed-953f-efc133f40ad0
        SIV:     286857941006449691324525859772784970448
decode: variant: DCE 1.1, ISO/IEC 11578:1996
        version: 5 (name based, SHA-1)
        content: D7:CE:D4:70:31:E7:04:ED:15:3F:EF:C1:33:F4:0A:D0
                 (not decipherable: truncated SHA-1 message digest only)
```

## UUIDv6

### IMPORTANT
This uuid version is still a draft & not fully added as standard!

Custom implementation [source](http://gh.peabody.io/uuidv6/)
```
import uuid

def uuidv1tov6(u):
  uh = u.hex
  tlo1 = uh[:5]
  tlo2 = uh[5:8]
  tmid = uh[8:12]
  thig = uh[13:16]
  rest = uh[16:]
  uh6 = thig + tmid + tlo1 + '6' + tlo2 + rest
  return uuid.UUID(hex=uh6)

u = uuidv1tov6(uuid.uuid1())
```

### python 2.x example
```bash
~$ python
>>> import uuid
>>> def uuidv1tov6(u):
...   uh = u.hex
...   tlo1 = uh[:5]
...   tlo2 = uh[5:8]
...   tmid = uh[8:12]
...   thig = uh[13:16]
...   rest = uh[16:]
...   uh6 = thig + tmid + tlo1 + '6' + tlo2 + rest
...   return uuid.UUID(hex=uh6)
...
>>> uuidv1tov6(uuid.uuid1())
UUID('1ebf6bb8-df40-6934-acf7-f875a4f5f762')
>>> uuidv1tov6(uuid.uuid1())
UUID('1ebf6bb9-16a3-617e-acf7-f875a4f5f762')
```

### python 3.x example
```bash
~$ python3
>>> import uuid
>>> def uuidv1tov6(u):
...   uh = u.hex
...   tlo1 = uh[:5]
...   tlo2 = uh[5:8]
...   tmid = uh[8:12]
...   thig = uh[13:16]
...   rest = uh[16:]
...   uh6 = thig + tmid + tlo1 + '6' + tlo2 + rest
...   return uuid.UUID(hex=uh6)
...
>>> uuidv1tov6(uuid.uuid1())
UUID('1ebf6bbf-44c0-6b28-acf7-f875a4f5f762')
>>> uuidv1tov6(uuid.uuid1())
UUID('1ebf6bbf-44c0-6b29-acf7-f875a4f5f762')
```

### Decode
As its still a draft, please visit [ietf.org](https://datatracker.ietf.org/doc/html/draft-peabody-dispatch-new-uuid-format) for more details.

TODO
Decoding as of 2021-08-06
```bash

```

## Sonyflake
is also a UUID but fits in an unsigned 64bit integer, so in terms of memory consumption & computing its faster than
all of them above

Used [sonyflake-py](https://pypi.org/project/sonyflake-py/) for the tests

### python 2.x example
```bash
~$ python

```

### python 3.x example
```bash
~$ python3
>>> import sonyflake
>>> from datetime import datetime, timezone
>>> sf = sonyflake.SonyFlake(machine_id=lambda: 1337, start_time=datetime(2021, 1, 1, 0, 0, 0, 0, tzinfo=timezone.utc))
>>> for x in range(5):
...     next = sf.next_id()
...     print(next)
...     print(sf.decompose(next))
...
31948301431473465
{'id': 31948301431473465, 'msb': 0, 'time': 1904267158, 'sequence': 0, 'machine_id': 1337}
31948301431539001
{'id': 31948301431539001, 'msb': 0, 'time': 1904267158, 'sequence': 1, 'machine_id': 1337}
31948301431604537
{'id': 31948301431604537, 'msb': 0, 'time': 1904267158, 'sequence': 2, 'machine_id': 1337}
31948301431670073
{'id': 31948301431670073, 'msb': 0, 'time': 1904267158, 'sequence': 3, 'machine_id': 1337}
31948301431735609
{'id': 31948301431735609, 'msb': 0, 'time': 1904267158, 'sequence': 4, 'machine_id': 1337}
```

### Decode
```python
from datetime import datetime, timezone, tzinfo
from sonyflake.sonyflake import SonyFlake


val = input("Enter id: ")
parts = SonyFlake.decompose(int(val))
start_time = SonyFlake.to_sonyflake_time(datetime(2021, 1, 1, 0, 0, 0, 0, tzinfo=timezone.utc))

print("DateTime: {}, MachineId: {}, Sequence: {}".format(
    datetime.fromtimestamp((parts['time'] + start_time) / 100),
    parts['machine_id'],
    parts['sequence']
))
```

If we enter `31948301431670073` the result will be
```bash
DateTime: 2021-08-09 11:37:51.580000, MachineId: 1337, Sequence: 2
```

### Conclusion
Contains the timestamp, which bot it produced and a sequence number. The time is in our case the important field, because we
are now able to sort events based upon time.
