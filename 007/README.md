# IEP007 - Running IntelMQ bots as Python Library

## Background
As of IntelMQ 3.1.0, IntelMQ Bots can only be started on the command line with `intelmqctl`.
Most tools (including the IntelMQ API, and thus the IntelMQ Manager) use `intelmqctl start` to start bot instances.
`intelmqctl start` spawns a new child process and detaches it.

Only `intelmqctl run` provides the ability to run bots interactively in the foreground of the command line and provides some neat features for debugging purposes.

Starting IntelMQ bots using Python code requires a lot of effort (and code complexity). Additionally, the bot's parameters can only be provided by modifying the IntelMQ runtime configuration file, and messages can only be fed and retrieved from the bot by connecting to the pipeline (e.g.) separately and writing/reading properly serialized messages there.

Minimal example in pseudo code:
```python
bot_instance = Bot(parameters)
bot_instance.process_message(input message) -> output messages
```

## Requirements

### Messages and Pipeline
It should be possible to provide input messages as function parameters and receiving output messages.

Messages should not be serialized or encoded, they should stay Message objects (derived from `dict` and behaving like dictionaries).

### Exceptions and dumped messages
An exception in the bot's `process()` method should not be catched in intermediate layers and raised to the caller's function call.

Option: If there is a helper function to call `process()` multiple times (having a bunch of input messages), the exceptions are caught together with the (dumped) messages, accessible to the caller.

### Parameters and Configuration
The global IntelMQ configuration should be effective.
The user may override configuration options by providing an configuration dictionary.

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
Global and bot configuration parameters are provided as paramter to the constructor in the same format as IntelMQ runtime configuration.

```python
class Bot:
    def __init__(bot_id: str,
                 *args, **kwargs,  # any other paramters left out for clarity
                 settings: Optional[dict] = None)
```
The constructor applies all values of the `settings` parameter after reading the runtime configuration file.

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
Should the processing continue and the exceptions be saved in a variable, which is returned at the end together with the sent messages?

## Examples

### A
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

### B
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
    # bot.process_call being a generator
    sent_messages = list(bot.process_message(EXAMPLE_REPORT))
    # sent_messages is now a list of sent messages
except IntelMQException as exc:
    sys.exit('Processing exception')
```
