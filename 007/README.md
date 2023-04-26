# IEP007 - Running IntelMQ bots as Python Library

A working example call (Proof of Concept) is located here:
https://github.com/wagner-intevation/intelmq/blob/bot-library/intelmq/tests/lib/test_bot.py#L141

## Background
As of IntelMQ 3.1.0, IntelMQ Bots can only be started on the command line with `intelmqctl`.
Most tools (including the IntelMQ API, and thus the IntelMQ Manager) use `intelmqctl start` to start bot instances.
`intelmqctl start` spawns a new child process and detaches it.

Only `intelmqctl run` provides the ability to run bots interactively in the foreground of the command line and provides some neat features for debugging purposes.

Starting IntelMQ bots using Python code requires much of effort (and code complexity). Additionally, the bot's parameters can only be provided by modifying the IntelMQ runtime configuration file.
Messages can only be fed and retrieved from the bot by connecting to the pipeline (e.g.) separately and writing/reading properly serialized messages there.

Integrating IntelMQ Bots into other (Python) tools is therefore hard to impossible in the current IntelMQ 3.1 version.

In a nutshell, calling a bot and processing should take, at most, a few lines.
The following complete example shows what the procedure could look like.
The bot class is instantiated, passing a few parameters.
```python
from intelmq.bots.experts.domain_suffix.expert import DomainSuffixExpertBot
domain_suffix = DomainSuffixExpertBot('domain-suffix',  # bot id
                                      settings={'logging_path': None,
                                                'source_pipeline_broker': 'Pythonlistsimple',  # TODO: simplify
                                                'destination_pipeline_broker': 'Pythonlistsimple',
                                                'field': 'fqdn',
                                                'suffix_file': '/usr/share/publicsuffix/public_suffix_list.dat',
                                                'destination_queues': {'_default': 'output',
                                                                       '_on_error': 'error'}})
queues = domain_suffix.process_message({'source.fqdn': 'www.example.com'})
# Select the output queue (as defined in `destination_queues`), first message, access the field 'source.domain_suffix':
# >>> output['output'][0]['source.domain_suffix']
# 'com'
```

### Use cases

#### General
Any IntelMQ-related or third-party program may use IntelMQ's most potent components - IntelMQ's bots.

The full potential shows off when stacking multiple bots together and iterating over lots of data:

```python
# instantiate all bots first, for an example see above
domain_suffix = DomainSuffixExpertBot(...)
url2fqdn = Url2fqdnExpertBot(...)
http_status = HttpstatusExpertBot(...)
tuency = TuencyExpertBot(...)
lookyloo = LookylooExpertBot(...)

# a list of input messages
messages = [{...}]

for message in message:
    for bot in (domain_suffix,
                url2fqdn,
                http_status,
                tuency,
                lookyloo):
        # for simiplicity we assume that the bots always send one message
        message = bot.process_message(message)['output'][0]
    # message now has the cumulated data of five bots

# messages now is a list of output messages
```

#### IntelMQ Webinput Preview

The IntelMQ webinput can show previews of the *processed* data to the operator, not just the input data, adding much more value to the preview functionality.
Currently the preview gives the operator feedback on the parsing step. The further processing of the data by the bots is invisible to the operator.
This causes confusion and uncertainty for the operators.

The Webinput backend can call the bots and process the events, without any interference to the running bot processes, pipelines and bot management.
The data flow illustrated:
```
Data provided by operator -> webinput backend parser -> IntelMQ bots as configured in the webinput configuration -> preview shown to operator
```
The implementation details for the webinput are not part of this proposal document.

In the next step, the webinput can also show previews of notifications (e.g. Emails). This is also not part of this proposal document.
```
Data provided by operator -> webinput backend parser -> IntelMQ bots as configured in the webinput configuration -> notification tool (preview mode) -> notification preview shown to operator
```

## Requirements

### Messages and Pipeline
Providing input messages as function parameters and receiving output messages should be possible.

Messages should not be serialized or encoded, they should stay Message objects (derived from `dict` and behaving like dictionaries).

### Exceptions and dumped messages
An exception in the bot's `process()` method should not be caught in intermediate layers and raised to the caller's function call.

Option: If there is a helper function to call `process()` multiple times (having a bunch of input messages), the exceptions are caught together with the (dumped) messages, accessible to the caller.

### Parameters and Configuration
The global IntelMQ configuration should be effective.
The user may override configuration options by providing a configuration dictionary.

#### Pre-configured bots
It should be possible to run bots defined in IntelMQ's runtime configuration file. Additional overriding parameters can be provided.

#### Un-Configured bots
It should be possible to run bots, which are not defined in IntelMQ's runtime configuration file. The bot configuration is provided as function parameter.

## Rationales

### Compatibility
Since the beginning of IntelMQ, the bot's `process` methods use the methods `self.receive_message`, `self.acknowledge_message` and `self.send_message`. Breaking this paradigm and changing to method parameters and return values or generator yields would indicate an API change and thus lead to IntelMQ version 4.0.
To be decided if this should be done.
Maybe as another IEP?

## Specification

Only changes in the `intelmq.lib.bot.Bot` class are needed.
No changes in the bots' code are required.

### Bot constructor

The operator constructs the bot by initializing the bot's class.
Global and bot configuration parameters are provided as parameter to the constructor in the same format as IntelMQ runtime configuration.

```python
class Bot:
    def __init__(bot_id: str,
                 *args, **kwargs,  # any other paramters left out for clarity
                 settings: Optional[dict] = None)
```
After reading the runtime configuration file, the constructor applies all values of the `settings` parameter.

### Method call

The `intelmq.lib.bot.Bot` class gets a new method `process_message`.
The definition:
```python
class Bot:
    def process_message(message: Optional[intelmq.lib.message.Message] = None):
```
For collectors:
    It takes *no* messages as input and returns a list of messages.
For parsers, experts and outputs:
    It takes exactly one message as input and returns a list of messages.
The messages are neither serialized nor encoded in any form, but are objects
of the `intelmq.lib.message.Message` class. If the message is of instance a dict
(with or without `__type` item), it will be automatically converted to the appropriate
Message object (`Report` or `Event`, depending on the Bot type).

Return value is a list of messages sent by the bot.
No exceptions of the bot are caught, the caller should handle them according to their needs.
The bot does not dump any messages to files on errors, irrelevant of the bot's dumping configuration.

As bots can send messages to multiple queues, the return value is a dictionary of all destination queues.
The items are lists, holding the sent messages.

#### Option: Processing multiple messages at once
This is a more complex situation in regards to error handling.
Should one exception stop the processing?
Should the processing continue and the exceptions be saved in a variable that is returned at the end with the sent messages?

## Examples

### Domain Suffix Expert Example

```python
from intelmq.bots.experts.domain_suffix.expert import DomainSuffixExpertBot
domain_suffix = DomainSuffixExpertBot('domain-suffix',  # bot id
                                      settings={'logging_path': None,
                                                'source_pipeline_broker': 'Pythonlistsimple',
                                                'destination_pipeline_broker': 'Pythonlistsimple',
                                                'field': 'fqdn',
                                                'suffix_file': '/usr/share/publicsuffix/public_suffix_list.dat',
                                                'destination_queues': {'_default': 'output',
                                                                       '_on_error': 'error'}})
queues = domain_suffix.process_message({'source.fqdn': 'www.example.com'})
# Select the output queue (as defined in `destination_queues`), first message, access the field 'source.domain_suffix':
# >>> output['output'][0]['source.domain_suffix']
# 'com'
```

### Accessing queues
```python
EXAMPLE_REPORT = {"feed.url": "http://www.example.com/",
                  "time.observation": "2015-08-11T13:03:40+00:00",
                  "raw": utils.base64_encode(RAW),
                  "__type": "Report",
                  "feed.name": "Example"}

bot = test_parser_bot.DummyParserBot('dummy-bot', settings={'global': {'logging_path': None,
                                                                       'source_pipeline_broker': 'Pythonlistsimple',
                                                                       'destination_pipeline_broker': 'Pythonlistsimple'},
                                                            'dummy-bot': {'parameters': {'destination_queues': {'_default': 'output',
                                                                                                                '_on_error': 'error'}}}})

sent_messages = bot.process_message(EXAMPLE_REPORT)
# sent_messages is now a dict with all queues. queue names below are examples

# this is the output queue
assert sent_messages['output'][0] == MessageFactory.from_dict(test_parser_bot.EXAMPLE_EVENT)
# this is a dumped message
assert sent_messages['error'][0] == input_message
```

### Option: bot.process_call is a generator
```python
from intelmq.lib.exceptions import IntelMQException

EXAMPLE_REPORT = {"feed.url": "http://www.example.com/",
                  "time.observation": "2015-08-11T13:03:40+00:00",
                  "raw": utils.base64_encode(RAW),
                  "__type": "Report",
                  "feed.name": "Example"}

bot = test_parser_bot.DummyParserBot('dummy-bot', settings={'global': {'logging_path': None,
                                                                       'source_pipeline_broker': 'Pythonlistsimple',
                                                                       'destination_pipeline_broker': 'Pythonlistsimple'},
                                                            'dummy-bot': {'parameters': {'destination_queues': {'_default': 'output',
                                                                                                                '_on_error': 'error'}}}})

try:
    # bot.process_call is a generator
    sent_messages = list(bot.process_message(EXAMPLE_REPORT))
    # sent_messages is now a list of sent messages
except IntelMQException as exc:
    sys.exit('Processing exception')
```
