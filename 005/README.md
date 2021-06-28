# IEP005: IntelMQ Data format: Notification settings

One of IntelMQ's core use-cases is to distribute warnings and IoC data to responsible parties like network or domain owners.
The distribution of data requires information of the delivery and recipient's address.

## Background

https://github.com/certtools/intelmq/issues/758

### Source of information

https://github.com/Intevation/intelmq-fody/
https://github.com/Intevation/intelmq-certbund-contact/

https://gitlab.com/Intevation/tuency/tuency
https://github.com/certtools/intelmq/pull/1857

### Delivering events

https://github.com/Intevation/intelmq-mailgen/tree/master/docs
https://github.com/certat/intelmq/blob/master/docs/user/intelmqcli.rst

### Requirements

#### Mail

For mail transfer, the following data fields are required:
- one or multiple destination email addresses
- optionally a PGP public key or fingerprint per address
- optionally an S/MIME certificate per address
- optionally a flag indicating, that the mail should be sent as *Cc:* to the recipient per address
- data format, e.g. CSV, JSON, IDEA, X-ARF, STIX etc.

Option 1: encryption information could be stored in a separate component, queried/used by the sending program.

#### HTTP/REST API

If the information is to be pushed to a foreign API, the following data fields are necessary:

- API endpoint, as URI, optionally containing username and password information
- optionally a client certificate
- optionally an authentication token

#### AMQP

- Hostname
https://www.rabbitmq.com/uri-spec.html
Does not support exchanges or queues, only vhost

#### XMPP

https://xmpp.org/rfcs/rfc5122.html#use-form

#### Generic

Timing interval information

#### Summary

|Delivery method|URI                  |Specific parameters|Authentication|PGP + S/MIME|Client Cert|Cc |Data format|
|---------------|---------------------|-------------------|--------------|------------|-----------|---|-----------|
|Mail           |One or more addresses|No                 |              |Yes         |No         |Yes|Yes        |
|XMPP           |Yes                  |Yes                |Yes           |No          |?          |No |Yes        |
|HTTP/REST API  |Yes                  |No                 |Yes           |No          |Yes        |No |Yes        |
|AMQP           |Yes                  |Yes                |Yes           |No          |Yes        |No |Yes        |


#### Option 2: Delivery references

Instead of replicating all information for every event, the events could as well hold only references. For PGP or S/MIME this could be the fingerprint, for AMQP/XMPP etc this could cover only connection parameters (authentication etc) without the host URI, or the host information as well.

## Proposal


