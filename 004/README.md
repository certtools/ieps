# IEP04 - Internal Data Format: Meta Information and Data Exchange


## Status
Decided and formalized via [JSON Schema](schema/schema.json)

## Contents
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

#### Variant AIL-A
Based on the [AIL Stream Format version 1](https://github.com/ail-project/ail-exchange-format/blob/main/ail-stream.md#user-content-ail-stream-format-version-1), mixed with original Variant A, see below:
```json
{
    "format": "intelmq", // or "n6" or "idea", so the receiving component can decode on demand.
    "version": 1, // protocol version, so we are allowed to fallback to old versions too
    "type": "event",
    "meta": {
       "intelmq:uuid": "event-uuid-1",
       "intelmq:uuid_org": "org-uuid", // the creating instance
       "intelmq:related": ["event-uuid-2", "event-uuid-3"],
       "intelmq:group": ["event-uuid-4"],
       "intelmq:alternate": ["RT#1234", "cesnet-certs:fed4740c-a8f7-11eb-9e47-efc1855d7a66"]
    },
    "payload": { // normal intelmq data
        "source.ip": "127.0.0.1",
        "source.fqdn": "example.com",
        "raw": // base64-blob
    }
}
```


#### Variant AIL-B
Based on the [AIL Stream Format version 1](https://github.com/ail-project/ail-exchange-format/blob/main/ail-stream.md#user-content-ail-stream-format-version-1), mixed with original Variant B, see below:
```json
{
    "format": "intelmq", // or "n6" or "idea", so the receiving component can decode on demand.
    "version": 1, // protocol version, so we are allowed to fallback to old versions too
    "type": "event",
    "meta": {
        "intelmq:uuid": "event-uuid-1",
        "intelmq:uuid_org": "org-uuid", // the creating instance
        "intelmq:links": [
            {
                "left_side": "event-uuid-2",
                "type": "is_parent_event",
                "right_side": "event-uuid-3"]
			},
			...
		]
    },
    "payload": { // normal intelmq data
        "source.ip": "127.0.0.1",
        "source.fqdn": "example.com",
        "raw": // base64-blob
    }
}
```


#### Variant A
Proposed by Pavel:
```json
{
    "meta": {
        "version": 1, // protocol version, so we are allowed to fallback to old versions too
        "uuid": {
            "origin": "org-uuid",
            "id": "event-uuid-1",
            "related": ["event-uuid-2", "event-uuid-3"],
            "group": ["event-uuid-4"],
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
Representing links in RDF, proposed by Aaron:
```json
{
    "meta": {
        "version": 1, // protocol version, so we are allowed to fallback to old versions too
        "uuid": {
            "origin": "org-uuids",
            "id": "event-uuid-1",
            "links": [
                {
                    "left_side": "event-uuid-2",
                    "type": "is_parent_event",
                    "right_side": "event-uuid-3"]
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

For the format of the UUID there are multiple options, see document [UUID](UUID.md) for a comparison.

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
