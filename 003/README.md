# IEP03 - Internal Data format: Multiple Values

Dear IntelMQ Developers and Users,

an evaluation of current challenges with the internal data format led to the idea of allowing multiple values for one field in IntelMQ 3.0 (scheduled for June 2021)[0].
The idea is described below, including various advantages and disadvantages. We appreciate your input, opinion and analysis of further implications on this idea.
We plan to evaluate the feedback that emerged in two weeks.

## Use-cases
### Network information
IntelMQ's format currently allows for *exactly one* value per field. For example, every event can have *one* `source.ip` and *one* `source.fqdn`. In some use-cases, multiple values can be useful, for example when querying DNS information. One domain (`source.fqdn`) can point to multiple IP addresses (`source.ip`). The other way round, multiple domains point to the same IP address is also very common. The use-case first appeared was that one IP address can be part of multiple Autonomous systems (`source.asn`).[1][2][3]

See the examples below in section Format.

### Classification
Another use-case is to use multiple classifications.[5] For example, if a website was hacked and used for a phishing page, it can be assigned two classifications:
For the hacking: Taxonomy: information-content-security, type: unauthorised-modification-of-information
For the phishing page: Taxonomy: fraud, type: phishing

Another example are reachable networks services, which should not be accessible by the internet. Shadowserver provides a lot of this data.
Open XDMCP instances are both DDoS amplifiers and Potentially unwanted accessible systems. Therefore both classifications apply:
Taxonomy: vulnerable, type: ddos-amplifier
Taxonomy: vulnerable, type: potentially-unwanted-accessible-system

A list of all fields on the RSIT can be found in the RSIT repository[6]

## Format
Some examples:
```json
{"source.ip": ["192.0.43.8"], "source.asn": [16876, 40528]}
{"source.ip": ["10.0.0.1", "10.0.0.2"], "source.url": ["http://example.com/", "http://example.net"]}
{"classification.taxonomy": ["information-content-security", "fraud"], "classification.type": ["unauthorised-modification-of-information", "phishing"], "source.url": ["http://example.com/"], "source.ip": ["10.0.0.1", "10.0.0.2"]}
```

In the bots' code multiple values need to be taken car of. For example, instead of:
```python
    ip_addr = event["source.ip"]
    # do stuff
```
it is necessary to loop over the values:
```python
    for ip_addr in event["source.ip"]:
        # do stuff
```
This logic is required for *all* fields which can have multiple values, therefore nested loops may be necessary.

Everything which processes IntelMQ data needs to be adapted, including data bases. See the "Disadvantages" section below.

### Optional back-conversion ("value-explosion")

One variant/option of this IEP is to create a conversion layer from the new multi-value format to the old one-value format by creating multiple events with only one value per field. Using this conversion, compatibility with external components can be kept, while the advantages only exist inside the IntelMQ core (ie. the bots).

Examples:
```json
{"source.ip": ["127.0.0.1", "127.0.0.2"], "source.fqdn": ["example.com"]}
    -> {"source.ip": "127.0.0.1", "source.fqdn": ["example.com"]}, {"source.ip": "127.0.0.2", "source.fqdn": ["example.com"]}
{"source.ip": ["127.0.0.1", "127.0.0.2"], "source.fqdn": ["example.com", "example.org"]}
    -> {"source.ip": "127.0.0.1", "source.fqdn": "example.com"}, {"source.ip": "127.0.0.1", "source.fqdn": "example.org"}, {"source.ip": "127.0.0.2", "source.fqdn": "example.com"}, {"source.ip": "127.0.0.2", "source.fqdn": "example.org"}
```

### What will change?
We'll change the behaviour of the current IntelMQ internal parsing process, i. e. you'll be able to add multiple IP addresses to on field, which will be handled as multiple events, but merged into one event.
This will allow us to combine i. e. a domain with multiple IP addresses to one event.

### Advantages

Supporting multiple values allows us to add multiple IP addresses to one event. As opposed to using multiple events with nearly similar data, the multiple-value approach reduces data duplication and has less overhead, while on the other hand the complexity increases.
If multiple events would be used instead, related events would need to be linked together by other means (see section Alternative below).

### Disadvantages (breaking behaviour)

The complexity in IntelMQ and all linked components increases without doubt. All components dealing with the IntelMQ-data need to be adapted to deal with multiple values. This includes all bots, but IntelMQ administrators need to adapt their configurations (e.g. filters, etc.) as well.

Without the explosion-variant, all connected databases need to be adapted (e.g. PostgreSQL, SQLite, Elastic, MongoDB etc.) additionally and all software which is processing data from IntelMQ need to be adapted. PostgreSQL support arrays for columns, but the scheme conversion can be complex and resource-hungry.

IntelMQ followed the KISS ("keep it simple, stupid")[4] principle from its beginning. It is disputable if multiple values breaks with this principle.

## Alternatives

An alternative to using multiple values per field is to set unique identifiers (e.g. UUID) per event and let events with the same origin have the same "parent" identifier. This way, related events can be linked and compatibility is easier. Relating the events to each other requires extra steps although, but keeps the KISS principle. This approach will be described in IEP04.

To solve the use-case of multiple classifications per event, the primary and most important classification can be used instead of multiple ones.

A possible solution for the classification use-case above would be to some sort of tagging - in short "tags". I. e.
```json
{
   "source.ip": ["192.0.43.8"],
   "source.asn": [16876, 40528],
   "tags": ["ddos-amplifier", "info-disclosure", "mirai-botnet"]
}
```

## Other IoC processing formats

For reference, we describe the formats of other IoC-processing systems similar to IntelMQ. Both formats, IDEA and n6 do support multiple values in different kinds. If you know of other similar formats supporting multiple values, please speak up!

### "IDEA"

The IDEA-format, used by CESNET-developed Warden, supports multiple values for some fields. But the data format structure differs clearly from IntelMQ's, as you can see in the example below. The classification is defined per address and network ranges are possible as addresses, what is not supported in IntelMQ.
IDEA was designed from scratch to overcome disadvantages of Warden's previous data format.

Example:
```json
   "Source": [
      {
         "Type": ["Phishing"],
         "IP4": ["192.168.0.2-192.168.0.5", "192.168.0.10/25"],
         "IP6": ["2001:0db8:0000:0000:0000:ff00:0042::/112"],
         "Hostname": ["example.com"],
         "URL": ["http://example.com/cgi-bin/killemall"],
         "Proto": ["tcp", "http"],
         "AttachHand": ["att1"],
         "Netname": ["ripe:IANA-CBLK-RESERVED1"]
      }
   ],
   "Target": [
      {
         "Type": ["Backscatter", "OriginSpam"],
         "Email": ["innocent@example.com"],
         "Spoofed": true
      },
      {
         "IP4": ["10.2.2.0/24"],
         "Anonymised": true
      }
   ]
```

Upstream documentation:
https://idea.cesnet.cz/en/index
https://warden.cesnet.cz/en/index

### n6

In the n6 format, the addr field is a list of arrays with `ip`, `asn`, `cc` and `dir` fields. `addr` is similar to IntelMQ's `source` namespace, but the size of `addr` is much lower and the "direction" of the address is given by a field inside the addr item.

Example:
```json
[{"ipv6": "abcd::1", "cc": "PL", "asn": 12345, "dir": "dst"}]
```

Upstream documentation:
https://n6sdk.readthedocs.io/en/latest/tutorial.html#field-class-addressfield
https://n6sdk.readthedocs.io/en/latest/tutorial.html#field-class-extendedaddressfield

## Links
[0]: https://github.com/certtools/intelmq/blob/version-3.0-ideas/docs/architecture-3.0.md#user-content-general-requirements
     "The new IDF shall support (sorted) lists of IPs, domains, taxonomy categories, etc. By convention the most relevant item in such a list MUST be the first item in the sorted list."
     https://github.com/certtools/intelmq/blob/version-3.0-ideas/docs/architecture-3.0.md#user-content-interoperability-with-certpls-[0]: n6-system
     "Since the new IDF shall support multiple values, mapping to n6 should be rather easy."
[1]: "Multiple ASNs/networks per IP? #543" https://github.com/certtools/intelmq/issues/543
[2]: "BOT: DNS lookup #373" https://github.com/certtools/intelmq/issues/373
[3]: "reverse DNS: Only first record is used "https://github.com/certtools/intelmq/issues/877
[4]: https://en.wikipedia.org/wiki/KISS_principle
[5]: https://github.com/enisaeu/Reference-Security-Incident-Taxonomy-Task-Force/blob/master/Documentation/Usage.md#user-content-multiple-classifications
[6]: https://github.com/enisaeu/Reference-Security-Incident-Taxonomy-Task-Force/blob/master/working_copy/humanv1.md