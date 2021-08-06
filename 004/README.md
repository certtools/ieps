# IEP04 - Internal Data Format: Meta Information and Data Exchange

Authors: Pavel Kacha (CESNET.cz), Sebastian Wagner (CERT.at), Sebastian Waldbauer (CERT.at), Aaron Kaplan (CERT.at)

To ease data exchange between two or more IntelMQ instances, adding some meta-information to the messages can make this sharing easier in certain regards.
For describing relations between messages ("links"), messages always have one UUID identifying themself and an arbitrary number of related UUIDs together with the link type.

The primary goal of this is to facilitate date-exchange between IntelMQ instance in different organisations, to prevent message loops.
The secondary goal is to express relations between messages.

## Possible Use-Cases for unique identifiers and message-to-message relations

The possible use-cases which can be solved by UUIDs per message:

1. Events with multiple target IPs/hostnames/ports
  - Horizontal portscan (multiple machines, one port)
  - SSH bruteforce (multiple machines, one port/service)
  - Vertical portscan (one machine, multiple ports)
2. Events with multiple source IPs/hostames/ports
  - Targeted DDoS (mutiple machines/reflectors shoot at one target)
3. Events with both multiple sources and targets
  - Wider DDoS (multiple machines/reflectors shoot at multiple machines,
    whole subnet, etc.)
4. Events with one or more both sources and targets, where exact pattern is
   not known
  - Aka one of [1, 2, 3], but we do not have complete information about
    specific connections made, possibly because the event/detection came
    from the statistical detector or from some form of aggregation (where
    original full information from for example netflow is already lost).

(1-4) initiated creation of IEP03 and IEP04 and are the ones considered.
Taking into account the possibility of linking of events, there might be other orthogonal use-cases:

5. Identification of identical events from possibly the same source to
   avoid duplication/circles
  - aka some form of stable identifier
6. When target organisation contacts source organisation for more info,
   identification of where event came from internally
  - aka possibility to put there the internal (opaque) identifier, like
    CESNET-RT#2235 (Request tracker), or Idea:UUID (what Idea event was
    converted into this IntelMQ event)
7. Meta-events
  - event, linking together multiple completely different events as one
    incident (email address of spammer from spam email, IPs of spamming
    mailservers, phishing URL from spam email)
8. Correlated events
  - aka different events, but identified as related/part of other events
    (like ongoing attack)
9. Modification or deletion/withdrawal of information
  - aka "this event replaces that event with new info", or "that event was
    wrong, sent by error, forget it"

I believe pretty much all are solvable by linking of events:
- 1, 2, 3 as bunch of linked events with source-target relation in each of them
- 4 as two linked events - one with all the sources, one with all the targets
- 5 as additional calculated identifier, hard part is not storage, but standardization/calculation
- 6 as additional opaque (freehand, non UUID) identifiers
- 7, 8 as bunch of linked events, with possibility of some meta-event maybe
- 9 as additional type of link

### Meta information
Metadata is used to transfer some general data, which is not likely related to the event itself. It's more or less just an information to keep events clear & sortable.

#### Variant A
A message could look like:
```json
{
    "meta": {
        "version": 1, // protocol version, so we are allowed to fallback to old versions too
        "uuid": {
           "origin": "" // the creating instance
           "id": "" // the id of the message itself
           "related": ["342a94d0-a8f6-11eb-b55d-3fbbafa57450", "d256379e-a8f7-11eb-8f63-2fb9e3acb5fb"],
           "group": ["ed4e10f8-a8f7-11eb-baaa-33efc701ab52"],
           "alternate": ["RT#1234", "cesnet-certs:fed4740c-a8f7-11eb-9e47-efc1855d7a66"]
        },
        "type": "event",
        "format": "intelmq", // or "n6" or "idea", so the receiving component can decode on demand.
    },
    "payload": { // normal intelmq data
        "source.ip": "127.0.0.1",
        "source.fqdn": "example.com",
        "raw": // base64-blob
    }
}
```

#### Variant B
Representing links in RDF:
```json
{
    "meta": {
        "version": 1, // protocol version, so we are allowed to fallback to old versions too
        "uuid": {
            "origin": "" // the creating instance
            "id": "" // the id of the message itself
            "links": [
                {
                    "left_side": "342a94d0-a8f6-11eb-b55d-3fbbafa57450",
                    "type": "is_parent_event",
                    "right_side": "d256379e-a8f7-11eb-8f63-2fb9e3acb5fb"]
                },
                ...
            ]
        },
        "type": "event",
        "format": "intelmq", // or: "n6" or "idea", so the receiving component can decode on demand.
    },
    "payload": { // normal intelmq data
        "source.ip": "127.0.0.1",
        "source.fqdn": "example.com",
        "raw": // base64-blob
    }
}
```

### The UUID
The purpose of the UUID is to identify the message uniquely.
The UUID is assigned upon creation of the message.

For the format of the UUID there are multiple options:
1. Generate a 128 bit (hex-string) UUIDv4.
2. Generate a [UUIDv6](http://gh.peabody.io/uuidv6/) ([RFC](https://datatracker.ietf.org/doc/html/draft-peabody-dispatch-new-uuid-format)). Smaller then UUIDv4, fits into a 64 bit integer, time-based, include the originator.
3. ~~A list of entities~~, which dealt with this event already. For example if an event was passed on from cert-at to cert-ee, the field could look like `!cert-at!cert-ee`. A message sending loop can be detected if the own name is already in this field upon reception.
4. Using CyCat: `publisher-short-name:project-short-name:UUID`. For example: `cert-at:intelmq:72ddb00c-2d0a-4eea-b7ac-ae122b8e6c3b`, or  `cert-pl:n6:f60c9fb9-81f9-4e0b-8a44-ea41326a15b3`. Some more research and discussion is required before the implementation of this option. Have a look at https://www.cycat.org/services/concept/ for more details.
5. ~~A hash~~: A benefit using a hash is that we're able to recalculate them on every intelmq instance. As this implies changed UUIDs on every minor change of the message, this approach has been dismissed.
6. [Sonyflake](https://github.com/certtools/ieps/issues/1#issuecomment-884079804-permalink): A compact UUID format using 39 bits for time in units of 10 msec, 8 bits for a sequence number, 16 bits for a machine (or bot) id. This covers the timestamp, the unique identity and origin.

### Exporting events to other systems
In IntelMQ 2.x the events only comprise of the "payload" and no meta information. For local storages like file output or databases, the meta information may not be relevant in some use-cases. So it needs to be possible to export events *without* meta information, which is also the backwards-compatible behaviour.

The "type" field exists in the current format as `__type` in the flat payload structure. In the output bots there's currently a boolean parameter `message_with_type` to include the field `__type` in the "export".
For optionally exporting meta-information like uuid or format, a similar logic could be used.

### Future extension of the Meta-Field
The Meta-field can be extended in the future by other fields, for example [FIRST IEP Policies](https://www.first.org/iep/iep-polices).
The Meta-field could be extendable by `x-*` fields. The usage of these fields should be documented in IntelMQ's format documentation.
This is not part of this IEP and will be specified in the future.

### Excursion: How can data exchange work?
This now depends on how IntelMQ instances can communicate, either Peer-to-peer or via a central data hub. Both of them do have pro's and con's.

#### P2P ( Peer 2 Peer )
Decentralized network
+ Less downtimes: A downtime of one instance, does not affect the whole network
+ Better privacy: data is not shared to an unrelated instance
+ More secure: data can optionally be encrypted (key-exchange between instances?)
+ Decentralized and local maintenance
~ Network latency depends on server locations
- Networking issues may occur

How would data exchange looks like between two instances:
1) Instance A has events which should be relayed to Instance B & C, because they're not sure who the actually receiver should be
2) Instance A ensures all messages have a UUID
3) Instance A sends the data to Instance B & Instance C
4) Instance B checks the data & they're sure that the data should be for Instance C
5) Instance C receives data from Instance A & Instance B
6) Instance C checks the UUID, which is the same & drops the package from Instance B

#### (Central) Data hub
+ Less maintenance: Is maintained by the hub administrator
+ Central data storage (reports can optionally be cached to be downloaded later)
~ Central data analysis (e.g. statistics) is possible
~ Network latency depends on server locations
- point of failure: if network problems occur, no exchange is possible

As already seen above, data exchange here would be less complicated. The sending may look like:
1) Instance A has events which should be relayed to Instance B (e.g. different country)
2) Instance A ensures all messages have a UUID
3) Instance A sends these messages to the data hub

The reception side can look like:
1) Instance B connects to central instance
2) Instance B queries and downloads all available messages
3) Upon reception, all messages are de-duplicated based on the UUID:
  a) If the UUID is already known, discard the message
  b) If the UUID has not been seen before, continue with processing

To sum up, both exchange variants are useful. More research is needed, i. e. a mixed infrastructure with centralized parts but can be decentralized too. However, this shall not be neither the purpose nor the aim of this IEP.

[0] https://github.com/certtools/intelmq/blob/version-3.0-ideas/docs/architecture-3.0.md#user-content-general-requirements
[1] https://github.com/certtools/intelmq/issues/1521
