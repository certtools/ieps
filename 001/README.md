# IEP01 - IntelMQ Configuration Handling

## Status

Implemented

## Format

### JSON

At the moment, the configuration format of IntelMQ is JSON. It is parsed using the Python `json` library, which is part of the [Python Standard Library](https://docs.python.org/3/library/index.html). The downside of JSON is, that is is hard to read and and write for humans and it cannot contain comments.

### YAML
There is a [proposal to use YAML](https://github.com/gethvi/intelmq/blob/ideas/docs/Ideas.md#changing-configuration-format-to-yaml) as the
default configuration format. YAML provides way better readability for humans and YAML supports single line comments. There are two Python YAML libraries out there, the one being [PyYAML](https://pyyaml.org/) and the other being [ruamel.yaml](https://yaml.readthedocs.io/en/latest/). The former is a project by the YAML project itself. The latter is a fork of the former and had much more activity over the years and better support of the standard. It seems that `pyyaml` caught up in the last few years.
We don't need any edge cases, so both libraries would be good for configuration files. According to [this issue](https://github.com/yaml/pyyaml/issues/46) `pyyaml` does not support "editing YAML whilst maintaining comments", which might be a deal breaker, but this issue is from 2016, this might have changed. On the other hand, IntelMQ does not **edit** configuration at the moment. `pyyaml` and `ruamel.yaml` are available as package in all relevant Linux distributions.

### INI

The Python Standard Library also ships [configparser](https://docs.python.org/3/library/configparser.html), which is a "configuration language
which provides a structure similar to what’s found in Microsoft Windows INI files". The files can contain comments, it comes with a `[DEFAULT]`
section, which can be used for default values and the configuration files can contain variables. One downside is that all the configurations are
Strings, which means we would have to do parsing ourself.

### toml

`Tom's Obvious, Minimal Language` is another contender for the role of IntelMQs configuration file format. It looks similar to the `INI` file
format, but comes with various data types. It also allows comments. There is a [Python library](https://pypi.org/project/toml/) that seems to
be very active. `toml` is also used as the format for the proposed `pyproject.toml` file and by the rust community for their package
configuration files. toml's syntax for dictionaries is hard to read/write, harder than with JSON.

## Further information

* [The summary on file formats on the PEP518 proposition](https://www.python.org/dev/peps/pep-0518/#other-file-formats)

* At the moment we are leaning towards YAML. Regarding the library, we would choose ruamel.yaml, because it seems to have a more active upstream and it can retain comments when it modifies a yaml file.

### Storage

This part is about the question `where do we store the configuration?`.

The [ideas document on GitHub](https://github.com/gethvi/intelmq/blob/ideas/docs/Ideas.md) already proposes to remove the `pipeline.conf` and
specifying the destination pipelines in the individual bot configuration part. The declaration of the source queue can be dropped then as well, as it follows a rule anyway.

In addition to that, to make the setup of IntelMQ easier, the `defaults.conf` should be dropped. Default values should be set in the `Bot`
classes respectively in the IntelMQ process managers, but there is no need for a separate file.

Another question is, if every bot should have their own configuration file. Some users wish to be able to start a bot without having to rely on
IntelMQ, but at the moment, the bot gets the configuration from IntelMQ's `runtime.conf`. If we want to support the request to be able to
pass individual configurations to bots, we could allow users to pass a separate configuration file to the bot (i.e. using `-c /path/to/config.$ext`).
If that file is not set or does not contain the bots id, it is ignored and IntelMQ's `runtime.conf` is used as usual. If it does exists, the
global `runtime.conf` is still parsed (if it exists - it should also be possible to run a bot without a `runtime.conf`) but only the values
that are not set in the individual configuration file are considered.
This `individual configuration file` would also allow a bot to be run in a docker environment without having to set any environment variables. This
would make configuration handling probably easier, because then configuration settings could be stored in a file (and managed by a configuration
management system) and the configuration file could contain comments.

# IntelMQ Enhancement Proposal (IEP):

  - IntelMQ gets **one** global configuration file for all the bots and the `pipeline.conf` will be removed
  - This global configuration file is `${PREFIX}/etc/intelmq/intelmq.$ext`. If it does not exists or does not define any bots, IntelMQ should exit gracefully. The file extension depends on the chosen format.
  - The global configuration file contains an array of bot configurations with `bot-id`s as keys.
  - Every bot reads the global configuration file and extracts their own settings (as usual).
  - Every bot handles `0` to `n` `-c /path/to/configurationfile.$ext` flags, which are treated the same way as the global configuration file. The further ahead the configuration file in the commandline, the stronger the content (this allows us to have multiple `non-global` configuration files (i.e. for multiple groups)) ``` Example: `> botcommand bot-id -c /etc/bots/botname.$ext -c /etc/bots/groups/group_foo.$ext```
  - Every bot also consults the environment and the values that are set their overwrite the values in any configuration file

  - There are also configuration files which list settings that are not bot specific, i.e. via a reserved key `default` (successor of the `defaults.conf` file) or `group:id`, those are also handled like other configuration files, but the bot does not compare its name to the key of the configuration.

All the evaluated configuration formats provide the possibility to arrange the configuration parameters in hierarchies. To make the configuration files more readable, IntelMQ should make use of this hierarchy instead of denoting the different hierarchy levels with underscores. So instead of writing `http_proxy` the `http` parameter would have a childparameter `proxy`. For backwards compatibility and cases where the underscore does not imply hierarchy, the underscore notation will still work. In addition, IntelMQ should also make use of environment variables - those are still denoted using an underscore as delimiter and are prepended with `INTELMQ`: `INTELMQ_HTTP_PROXY`.

## Caveats

There are configuration settings, that do not really concern the bot- for example the type of process manager, that should be used to run the bot.
In an ideal setup, the bot should be totally indifferent as to if it runs in a Docker container, on bare metal, in a SystemD unit file or with
SupervisorD. This decision should only concern the tool managing all the bots (`intelmqctl` or in the future `intelmq-api` (which at the moment
uses `intelmqctl`)). Another example is the `enabled` setting.
At the moment, those are part of the individual bot configuration, but it might make sense to move them to a `management.conf` configuration file
which is **only** for managing the individual bots, but not for configuring their parameters (this file would then also (for every bot) have a field
that lists the configuration files the bot should consider when reading its configuration). On the other hand, this might make the configuration
more complex again, now that we are trying to merge `pipeline.conf` and `runtime.conf`. We could also decide to make those configuration
settings be part of the global configuration file, given that the individual bots should anyway simply ignore settings they do not know how to handle.

## Overriding by command line parameters

If needed, a user can override specific bot settings using the `-p` switch (i.e. `-p redis_cache=example.com`). This should be easy to implement,
in the best case scenario this is only one line of additional code in the `Bot` class.

## Examples

A global configuration file with multiple bots `/etc/intelmq/intelmq.yml`
```yaml
- shodan1:
    module: intelmq.bots.collectors.shodan.collector
- mylittlebot23:
    module: intelmq.bots.expert.asn_lookup.expert
    http:
      proxy: http://myproxy.tld:80
- fop1:
    module: intelmq.bots.outputs.file
    output:
      filename: /dev/null
```

We can run a bot with `intelmq-bot shodan1` which is the same is `intelmq-bot shodan1 -c /etc/intelmq/intelmq.yml`

Another configuration file with multiple bots `/root/intelmq-bots-managed-by-root`:

```yaml
- shodan2:
    module: intelmq.bots.collectors.shodan.collector
- fop1:
    module: intelmq.bots.outputs.file
    output:
      filename: /var/log/fop1.log
```

We can run a bot with `intelmq-bot shodan2 -c /root/intelmq-bots-managed-by-root`; We can run a bot using `intelmq-bot fop1 -c /root/intelmq-bots-managed-by-root` which would then send output to `/var/log/fop1.log`.

A configuration for a group in `/etc/intelmq/collector-group.yml`

```yaml
- group:collectors
  http:
    proxy: http://thirdparty.proxy.tld:9000
```

We can run a bot with `intelmq-bot mylittlebot23 -c /etc/intelmq/collector-group.yml` which uses the third-party proxy.

## Internal handling

Every bot class defines their own settings as class variables.
Every class variable has to be typed.
Every class variable should be set to a reasonable default, otherwise `None`.
The `__init__` of the (abstract) `Bot` class should load all the relevant configuration files and then overwrite the settings.
If a setting is still `None` and the value of the setting is vital for the functionality of the bot, the bot should stop
and emit a meaningful error message.
For the most common types of settings, there should be Python objects to check the values.
Value checking should only be done after all the configurations are merged.
