Configuration
=============

A Benthos stream is configured either in a YAML or JSON file using a
hierarchical format. For a basic stream pipeline that means the configuration
is very simple:

``` yaml
input:
  type: kafka
  kafka:
    topic: foo
    partition: 0
    addresses:
      - localhost:9092
output:
  type: file
  file:
    path: ./foo.txt
```

However, as configurations become more complex and hierarchical this format can
sometimes be difficult to read and manage:

``` yaml
input:
  type: kafka
  kafka:
    ...
  processors:
    - type: conditional
      conditional:
        condition:
          type: jmespath
          jmespath:
            query: contains(foo.bar, "compress me")
        processors:
          - type: compress
            compress:
              algorithm: gzip
```

The above example reads messages from Kafka and, if the JSON path `foo.bar`
contains the phrase "compress me" the entire message will be compressed with
`gzip`, otherwise it passes unchanged. This isn't complicated behaviour but
there are many ways in which this configuration could be broken, both during
conception and when being modified at a later date.

This document outlines tooling provided by Benthos to help with writing and
managing these more complex configuration files, as well as some generally good
practices to follow.

## Contents

- [Helping With Discovery](#helping-with-discovery)
- [Helping With Debugging](#helping-with-debugging)
- [Good Practices](#good-practices)

## Helping With Discovery

The discoverability of configuration fields is a common headache with any
configuration driven application. The classic solution is to provide curated
documentation that is often hosted on a dedicated site. Benthos does this by
generating a markdown document
[per configuration section](./README.md#core-components).

However, a user often only needs to get their hands on a short, runnable example
config file for their use case. The fields themselves are usually self
explanatory, they just need to see the format and field names. Forcing such a
user to navigate a website, scrolling through paragraphs of text, seems
inefficient when all they actually needed to see was something like:

``` yaml
input:
  type: amqp
  amqp:
    url: amqp://guest:guest@localhost:5672/
    consumer_tag: benthos-consumer
    exchange: benthos-exchange
    exchange_type: direct
    key: benthos-key
    prefetch_count: 10
    prefetch_size: 0
    queue: benthos-queue
output:
  type: stdout
```

In order to make this process easier Benthos generates usable configuration
examples in the [config directory](../config). There is a config file for each
input and output type, and inside the [processors](../config/processors) sub
directory there is a file showing each processor type, and so on.

If you are a user interested in sending MQTT messages to Kafka you need only
make a copy of `config/kafka.yaml`, copy some lines from `config/mqtt.yaml`, and
you have a runnable config file. The file will also include other useful config
sections such as `metrics`, `logging`, etc with sensible defaults.

### Printing Every Field

The format of a Benthos config file naturally exposes all of the options for a
section when it's printed with all default values. For example, in a fictional
section `foo`, which has type options `bar`, `baz` and `qux`, if you were to
print the entire default `foo` section of a config it would look something like
this:

``` yaml
foo:
  type: bar
  bar:
    field1: default_value
    field2: 2
  baz:
    field3: another_default_value
  qux:
    field4: false
```

Which tells you that section `foo` supports the three object types `bar`, `baz`
and `qux`, and defaults to type `bar`. It also shows you the fields that each
section has, and their default values.

The Benthos binary is able to print a JSON or YAML config file containing every
section in this format with the commands `benthos --print-yaml --all` and
`benthos --print-json --all`. This can be extremely useful for quick and dirty
config discovery when the full repo isn't at hand.

As a user you could create a new config file with:

``` sh
benthos --print-yaml --all > conf.yaml
```

And simply delete all lines for sections you aren't interested in, then you are
left with the full set of fields you want. Alternatively, using tools such as
`jq` you can extract specific type fields:

``` sh
# Get all Kafka input fields:
benthos --print-json --all | jq '.input.kafka'

# Get all AMQP output fields:
benthos --print-json --all | jq '.output.amqp'

# Get all JSON processor fields:
benthos --print-json --all | jq '.pipeline.processors[0].json'
```

## Helping With Debugging

Once you have a config written you now move onto the next headache of proving
that it works, and understanding why it doesn't. Benthos, like most good config
driven services, performs validation on configs and tries to provide sensible
error messages.

However, with validation it can be hard to capture all problems, and the user
usually understands their intentions better than the service. In order to help
expose and diagnose config errors Benthos has two ways of echoing back your
configuration _after_ it has been parsed.

The first is with the `--print-yaml` and `--print-json` commands, which print
the Benthos configuration in YAML and JSON format respectively, this is done
_after_ parsing and applying your config. The result is that Benthos can show
you exactly how your config was interpretted:

``` sh
benthos -c ./your-config.yaml --print-yaml
```

You can check the output of the above command to see if sections are missing or
fields are incorrect, which allows you to pinpoint typos in the config.

If your configuration is complex, and the behaviour that you notice implies a
certain section is at fault, then you can drill down into that section by using
tools such as `jq`:

``` sh
# Check the second processor config
benthos -c ./your-config.yaml --print-json | jq '.pipeline.processors[1]'

# Check the condition of a filter processor
benthos -c ./your-config.yaml --print-json | jq '.pipeline.processors[0].filter'
```

## Good Practices

TODO