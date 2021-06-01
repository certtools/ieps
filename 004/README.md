# IEP04 - Internal Data Format: Meta Information and Data Exchange

Authors: Pavel Kacha (CESNET.cz), Sebastian Wagner (CERT.at), Sebastian Waldbauer (CERT.at)

To ease data exchange between two or more IntelMQ instances, adding some meta-information to the messages can make this sharing easier in certain regards.
For describing relations between messages ("links"), messages always have one UUID identifying themself and an arbitrary number of related UUIDs together with the link type.

## TL;DR
Communication between one or more IntelMQ instances & exchange data with a backwards-compatible format. P2P or centralized architecture is a big topic, which has to be discussed after the format is being set.

## Use-Cases for unique identifiers and message-to-message relations

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
1, 2, 3 as bunch of linked events with source-target relation in each of them
4 as two linked events - one with all the sources, one with all the targets
5 as additional calculated identifier, hard part is not storage, but standardization/calculation
6 as additional opaque (freehand, non UUID) identifiers
7, 8 as bunch of linked events, with possibility of some meta-event maybe
9 as additional type of link

### Meta information
Metadata is used to transfer some general data, which is not likely related to the event itself. It's more or less just an information to keep events clear & sortable.

A message could look like:
```json
{
    "meta": {
        "version": 1, // protocol version, so we are allowed to fallback to old versions too
        "uuid": {
           "current": "cert_at:aaaa-bbbb-cccc-dddd" // format to be decided
           "parent": "cert_at:xxxx-yyyy-zzzz-ffff" // format to be discussed, if not set -> current is the parent uuid
        },
        "type": "event",
        "format": "intelmq", // i. e. this field could contain "n6" or "idea", so the receiving component can decode on demand.
    },
    "payload": { // normal intelmq data
        "source.ip": "127.0.0.1",
        "source.fqdn": "example.com",
        "raw": // base64-blob
    }
}
```

Tell us your opinion about adding non-standardized meta-information fields ( i. e. RTIR ticket number, origin, other local contact informationen ... and so on )

#### The UUID
For the UUID there are multiple options:
1. Generate a random 128 bit UUID
2. A list of entities, which dealt with this event already. For example if an event was passed on from cert-at to cert-ee, the field could look like `!cert-at!cert-ee`. A message sending loop can be detected if the own name is already in this field upon reception.
3. Using CyCat: `publisher-short-name:project-short-name:UUID`. For example: `cert-at:intelmq:72ddb00c-2d0a-4eea-b7ac-ae122b8e6c3b`, or  `cert-pl:n6:f60c9fb9-81f9-4e0b-8a44-ea41326a15b3`. Some more research and discussion is required before the implementation of this option. Have a look at https://www.cycat.org/services/concept/ for more details.
4. A hash: A benefit using a hash is that we're able to recalculate them on every intelmq instance.

### Exporting events to other systems
In IntelMQ 2.x the events only comprise of the "payload" and no meta information. For local storages like file output or databases, the meta information may not be relevant in some use-cases. So it needs to be possible to export events *without* meta information, which is also the backwards-compatible behaviour.

The "type" field exists in the current format as `__type` in the flat payload structure. In the output bots there's currently a boolean parameter `message_with_type` to include the field `__type` in the "export".
For optionally exporting meta-information like uuid or format, a similar logic could be used.

### How can data exchange work?
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
