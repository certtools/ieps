# IEP04 - Internal Data Format: Meta Information and Data Exchange
To ease data exchange between two or more IntelMQ instances, adding some meta-information to the events can make this sharing easier in certain regards.
"Linking" events could be based on the same theory as `git` using it - with parent hashes ( we would call it UUID ).

### TL;DR
Communication between one or more IntelMQ instances & exchange data with a backwards-compatible format. P2P or centralized architecture is a big topic, which has to be discussed after the format is being set.

### Why is metadata important?
Short and simple. To avoid race conditions & being able to discard/drop already processed events from other instances.

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

The "type" field exists in the current format as "__type" in the flat payload structure. In the output bots there's currently a boolean parameter `message_with_type` to include the field `__type` in the "export".
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