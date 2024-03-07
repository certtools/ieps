# IEP008: IntelMQ Data Format: Constituency field

At present, an event message describes the affected parties, but does not contain any information
about logical grouping of them, which may be important to the system operator. Today, more and more
teams act as CSIRT/CERTs for multiple constituencies and therefore may need to handle some parts of the workflow differently.

This proposal introduces an additional field in the IDF to handle information about the
related constituency.

## Background

Support for constituencies is already spreading throughout the IntelMQ world, for example the Tuency
API is designed to return tenant names as constituencies in the API for integration with IntelMQ
[[1]](https://gitlab.com/intevation/tuency/tuency/-/blob/master/backend/docs/IntelMQ-API.md#data-structure-of-the-result).

## Use cases

Consider the workflow where Tuency or another bot (e.g. Sieve) sets the correct constituency field
depending on the source concerned. Then the following use cases could be easier:

  * the part responsible for sending notifications can choose the right communication channel
  (e.g. the right ticketing system, outgoing email address, etc.);
  * They can filter out or escalate certain types of events depending on the constituency affected.

## Solution considerations

It is possible to create a more or less restrictive solution to the problem.

The more restrictive solution would be
  * Define a strongly validated field that only allows certain values (enum),
  * Allow the operator to configure the supported constituencies at the IntelMQ level.

To achieve this, the IDF would need to be modified, as well as IntelMQ itself, since the current
configuration is focused on bots and not on changing IDF field values. On the other hand,
it prevents the possibility of inadvertently configuring bots with the wrong names without noticing.

The less restrictive solution is to simply add a string field to the IDF. With this approach
implementation is transparent for single constituency setups, and in more complex cases, all
validation can still be done using additional bots, e.g. Sieve or Filter experts.

There is also the question of whether we should allow one or more constituencies to be defined
in the event. However, the philosophy of IntelMQ is strongly oriented towards processing single
pieces of information. Assigning multiple constituencies to an event would, in my opinion, break this
philosophy, and the proposal of multiple values has already been rejected [[2]](https://github.com/certtools/ieps/tree/main/003).

Therefore, I suggest that for cases where multiple assignments are possible, the IntelMQ operator
should configure the workflow to create or save the event multiple times with multiple
and finally define a way to correlate them.

## Proposal

Because of the simplicity, I've chosen to propose adding a simple text field in the IDF, defined as:

```
"constituency": {
    "description": "Internal identifier for multi-constituency setup",
    "type": "String"
}
```

This should only be added to the `event` schema.
