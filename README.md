# How to use the Druid Data Driver

## Program description

The Druid Data Driver is a python script that simulates a workload that generates data for Druid ingestion.
You can use a JSON config file to describe the characteristics of the workload you want the Druid Data Driver to simulate.
The script uses this config file to generate JSON records.

Here are the commands to set up the Python environment:

```
apt-get install python3
apt-get update
apt-get install -y python3-pip

pip install confluent-kafka
pip install python-dateutil
pip install kafka-python
pip install numpy
pip install sortedcontainers
```

Run the program as follows:

```python DruidDataDriver.py <options>```

Options include:

```
-f <configuration file name>
-s
-n <total number of records to generate>
-t <duration for generating records>
```

Use the _-f_ option to designate a configuration file name.
If you omit the _-f_ option, the script reads the configuration from _stdin_.

The _-s_ option tells the driver to use simulated time instead of wall clock time (the default).
The simulated clock starts at the current time and advances the simulated clock based on the
generated events (i.e., records).
When used with the _-t_ option, the simulated clock simulates the duration.
This option is useful for generating batch data as quickly as possible.

The other two options control how long the script runs.
If neither option is present, the script will run indefinitely.
The _-t_ and _-n_ options are exclusive (use one or the other).
Time durations may be specified in terms of seconds, minutes or hours.
For example, specify 30 seconds as follows:

```
-t 30S
```

Specify 10 minutes as follows:

```
-t 10M
```

Or, specify 1 hour as follows:

```
-t 1H
```

## Config File

The config file contains JSON describing the characteristics of the workload you want to simulate (see the _examples_ folder for example config files).
A workload consists of a state machine, where each state outputs a record.
The state machine is probabilistic, which means that the state transitions may be stochastic based on probabilities.
Each state in the state machine performs four operations:
- First, the state sets any variable values
- Next, the state emits a record (based on an emitter description)
- The state delays for some period of time (based on a distribution)
- Finally, the state selects and transitions to a different state (based on a probabilistic transition table)

Emitters are record generators that output records as specified in the emitter description.
Each state employs a single emitter, but the same emitter may be used by many states.

The config file has the following format:

```
{
  "target": {...},
  "emitters": [...],
  "interarrival": {...},
  "states": [...]
}
```

The _target_ object describes the output destination.
The _emitters_ list is a list of record generators.
The _interarrival_ object describes the inter-arrival times (i.e., inverse of the arrival rate) of entities to the state machine
The _states_ list is a description of the state machine

### Distribution descriptor objects

Use _distribution descriptor objects_ to parameterize various characteristics of the config file (e.g., inter-arrival times, dimension values, etc.) according to the config file syntax described in this document.

There are four types of _distribution descriptor objects_: Constant, Uniform, Exponential and Normal. Here are the formats of each of these types:

#### Constant distributions

The constant distribution generates the same single value.

```
{
  "type": "constant",
  "value": <value>
}
```

Where _value_ is the value generated by this distribution.

#### Uniform distribution

Uniform distribution generates values uniformly between _min_ and _max_ (inclusive).

```
{
  "type": "uniform",
  "min": <value>,
  "max": <value>
}
```

Where:
- _min_ is the minimum value sampled
- _max_ is the maximum value sampled

#### Exponential distribution

Exponenital distributions generate values following an exponential distribution around the mean.

```
{
  "type": "exponential",
  "mean": <value>
}
```

Where _mean_ is the resulting average value of the distribution.

#### Normal distribution

Normal distributions generate values with a normal (i.e., bell-shaped) distribution.

```
{
  "type": "normal",
  "mean": <value>,
  "stddev": <value>
}

```

Where:
- _mean_ is the average value
- _stddev_ is the stadard deviation of the distribution

Note that negative values generated by the normal distribution may be forced to zero when necessary (e.g., interarrival times).

### "target": {}

There are four flavors of targets: _stdout_, _file_, _kafka_, and _confluent_.

_stdout_ targets print the JSON records to standard out and have the form:

```
"target": {
  "type": "stdout"
}
```

_file_ targets write records to the specified file and have the following format:

```
"target": {
  "type": "file",
  "path": "<filename goes here>"
}
```

Where:
- <i>path</i> is the path and file name

_kafka_ targets write records to a Kafka topic and have this format:

```
"target": {
  "type": "kafka",
  "endpoint": "<ip address and optional port>",
  "topic": "<topic name>",
  "security_protocol": "<protocol designation>",
  "compression_type": "<compression type designation>"
}
```

Where:
- <i>endpoint</i> is the IP address and optional port number (e.g., "127.0.0.1:9092") - if the port is omitted, 9092 is used
- <i>topic</i> is the topic name as a string
- <i>security_protocol</i> (optional) a protocol specifier ("PLAINTEXT" (default if omitted), "SSL", "SASL_PLAINTEXT", "SASL_SSL")
- <i>compression_type</i> (optional) a compression specifier ("gzip", "snappy", "lz4") - if omitted, no compression is used

_confluent_ targets write records to a Confluent topic and have this format:

```
"target": {
  "type": "confluent",
  "servers": "<bootstrap servers>",
  "topic": "<topic name>",
  "username": "<username>",
  "password": "<password>"
}
```

Where:
- <i>servers</i> is the confluent servers (e.g., "pkc-lzvrd.us-west4.gcp.confluent.cloud:9092")
- <i>topic</i> is the topic name as a string
- <i>username</i> cluster API key
- <i>password</i> cluster API secret


### "emitters": []

The _emitters_ list is a list of record generators.
Each emitter has a name and a list of dimensions, where the list of dimensions describes the records the emitter will generate.

An example of an emitter list looks as follows:

```
"emitters": [
  {
    "name": "short-record",
    "dimensions": [...]
  },
  {
    "name": "long-record",
    "dimensions": [...]
  }
]
```

#### "dimensions": []

The _dimensions_ list contains specifications for all dimensions (except the <i>__time</i> dimension, which is always the first dimension in the output record specified in UTC ISO 8601 format, and has the value of when the record is actually generated).

##### Cardinality

Many dimension types may include a _Cardinality_ (the one exception is the _enum_ dimension type).
Cardinality defines how many unique values the driver may generate.
Setting _cardinality_ to 0 provides no constraint to the number of unique values.
But, setting _cardinality_ > 0 causes the driver to create a list of values, and the length of the list is the value of _cardinality_.

When _cardinality_ is greater than 0, <i>cardinality_distribution</i> informs how the driver selects items from the cardinality list.
We can think of the cardinality list as a list with zero-based indexing, and the <i>cardinality_distribution</i> determines how the driver will select an index into the cardinality list.
After using the <i>cardinality_distribution</i> to produce an index, the driver constrains the index so as to be a valid value (i.e., 0 <= index < length of cardinality list).
Note that while _uniform_ and _normal_ distributions make sense to use as distribution specifications, _constant_ distributions only make sense if the cardinality list contains only a single value.
Further, for cardinality > 0, avoid an _exponential_ distribution as it will round down any values that are too large and produces a distorted distribution.

As an example of cardinality, imagine the following String element definition:

```
{
  "type": "string",
  "name": "Str1",
  "length_distribution": {"type": "uniform", "min": 3, "max": 6},
  "cardinality": 5,
  "cardinality_distribution": {"type": "uniform", "min": 0, "max": 4},
  "chars": "abcdefg"
}
```

This defines a String element named Str1 with five unique values that are three to six characters in length.
The driver will select from these five unique values uniformly (by selecting indices in the range of [0,4]).
All values may only consist of strings containing the letters a-g.

##### Dimension list item types

Dimension list entries include:

###### { "type": "enum" ...}

Enum dimensions specify the set of all possible dimension values, as well as a distribution for selecting from the set.
Enums have the following format:

```
{
  "type": "enum",
  "name": "<dimension name>",
  "values": [...],
  "cardinality_distribution": <distribution descriptor object>,
  "percent_missing": <percentage value>,
  "percent_nulls": <percentage value>
}
```

Where:
- <i>name</i> is the name of the dimension
- <i>values</i> is a list of the values
- <i>cardinality_distribution</i> informs the cardinality selection of the generated values
- <i>percent_missing</i> a value in the range of 0.0 and 100.0 (inclusive) indicating the stochastic frequency for omitting this dimension from records (optional - the default value is 0.0 if omitted)
- <i>percent_nulls</i> a value in the range of 0.0 and 100.0 (inclusive) indicating the stochastic frequency for generating null values (optional - the default value is 0.0 if omitted)


###### { "type": "string" ...}

String dimension speficiation entries have the following format:

```
{
  "type": "string",
  "name": "<dimension name>",
  "length_distribution": <distribution descriptor object>,
  "cardinality": <int value>,
  "cardinality_distribution": <distribution descriptor object>,
  "chars": "<list characters used to build strings>",
  "percent_missing": <percentage value>,
  "percent_nulls": <percentage value>
}
```

Where:
- <i>name</i> is the name of the dimension
- <i>length_distribution</i> describes the length of the string values - Some distribution configurations may result in zero-length strings
- <i>cardinality</i> indicates the number of unique values for this dimension (zero for unconstrained cardinality)
- <i>cardinality_distribution</i> informs the cardinality selection of the generated values (omit if cardinality is zero)
- <i>chars</i> (optional) is a list (e.g., "ABC123") of characters that may be used to generate strings - if not specified, all printable characters will be used
- <i>percent_missing</i> a value in the range of 0.0 and 100.0 (inclusive) indicating the stochastic frequency for omitting this dimension from records (optional - the default value is 0.0 if omitted)
- <i>percent_nulls</i> a value in the range of 0.0 and 100.0 (inclusive) indicating the stochastic frequency for generating null values (optional - the default value is 0.0 if omitted)

##### { "type": "int" ...}

Integer dimension specification entries have the following format:

```
{
  "type": "int",
  "name": "<dimension name>",
  "distribution": <distribution descriptor object>,
  "cardinality": <int value>,
  "cardinality_distribution": <distribution descriptor object>,
  "percent_missing": <percentage value>,
  "percent_nulls": <percentage value>
}
```

Where:
- <i>name</i> is the name of the dimension
- <i>distribution</i> describes the distribution of values the driver generates (rounded to the nearest int value)
- <i>cardinality</i> indicates the number of unique values for this dimension (zero for unconstrained cardinality)
- <i>cardinality_distribution</i> skews the cardinality selection of the generated values
- <i>percent_missing</i> a value in the range of 0.0 and 100.0 (inclusive) indicating the stochastic frequency for omitting this dimension from records (optional - the default value is 0.0 if omitted)
- <i>percent_nulls</i> a value in the range of 0.0 and 100.0 (inclusive) indicating the stochastic frequency for generating null values (optional - the default value is 0.0 if omitted)

###### { "type": "float" ...}

Float dimension specification entries have the following format:

```
{
  "type": "float",
  "name": "<dimension name>",
  "distribution": <distribution descriptor object>,
  "cardinality": <int value>,
  "cardinality_distribution": <distribution descriptor object>,
  "percent_missing": <percentage value>,
  "percent_nulls": <percentage value>,
  "precision": <number of digits after decimal>
}
```

Where:
- <i>name</i> is the name of the dimension
- <i>distribution</i> describes the distribution of float values the driver generates
- <i>cardinality</i> indicates the number of unique values for this dimension (zero for unconstrained cardinality)
- <i>cardinality_distribution</i> skews the cardinality selection of the generated values
- <i>percent_missing</i> a value in the range of 0.0 and 100.0 (inclusive) indicating the stochastic frequency for omitting this dimension from records (optional - the default value is 0.0 if omitted)
- <i>percent_nulls</i> a value in the range of 0.0 and 100.0 (inclusive) indicating the stochastic frequency for generating null values (optional - the default value is 0.0 if omitted)
- <i>precision</i> (optional) the number digits after the decimal - if omitted all digits are included

###### { "type": "timestamp" ...}

Timestamp dimension specification entries have the following format:

```
{
  "type": "timestamp",
  "name": "<dimension name>",
  "distribution": <distribution descriptor object>,
  "cardinality": <int value>,
  "cardinality_distribution": <distribution descriptor object>,
  "percent_missing": <percentage value>,
  "percent_nulls": <percentage value>
}
```

Where:
- <i>name</i> is the name of the dimension
- <i>distribution</i> describes the distribution of timestamp values the driver generates
- <i>cardinality</i> indicates the number of unique values for this dimension (zero for unconstrained cardinality)
- <i>cardinality_distribution</i> skews the cardinality selection of the generated timestamps (optional - omit for unconstrained cardinality)
- <i>percent_missing</i> a value in the range of 0.0 and 100.0 (inclusive) indicating the stochastic frequency for omitting this dimension from records (optional - the default value is 0.0 if omitted)
- <i>percent_nulls</i> a value in the range of 0.0 and 100.0 (inclusive) indicating the stochastic frequency for generating null values (optional - the default value is 0.0 if omitted)


###### { "type": "ipaddress" ...}

IP address dimension specification entries have the following format:

```
{
  "type": "ipaddress",
  "name": "<dimension name>",
  "distribution": <distribution descriptor object>,
  "cardinality": <int value>,
  "cardinality_distribution": <distribution descriptor object>,
  "percent_missing": <percentage value>,
  "percent_nulls": <percentage value>
}
```

Where:
- <i>name</i> is the name of the dimension
- <i>distribution</i> describes the distribution of IP address values the driver generates
- <i>cardinality</i> indicates the number of unique values for this dimension (zero for unconstrained cardinality)
- <i>cardinality_distribution</i> skews the cardinality selection of the generated IP addresses (optional - omit for unconstrained cardinality)
- <i>percent_missing</i> a value in the range of 0.0 and 100.0 (inclusive) indicating the stochastic frequency for omitting this dimension from records (optional - the default value is 0.0 if omitted)
- <i>percent_nulls</i> a value in the range of 0.0 and 100.0 (inclusive) indicating the stochastic frequency for generating null values (optional - the default value is 0.0 if omitted)


Note that the data driver generates IP address values as <i>int</i>s according to the distribution, and then converts the int value to an IP address.

###### { "type": "variable" ...}

Variables are values that may be set by states and have the following format:

```
{
  "type": "variable",
  "name": "<dimension name>"
  "variable": "<name of variable>"
}
```

Where:
- <i>name</i> is the name of the dimension
- <i>variable</i> is the name of variable with a previously set value

###### { "type": "object" ...}

Object dimensions create nested data. Object dimension specification entries have the following format:

```
{
  "type": "object",
  "name": "<dimension name>",
  "cardinality": <int value>,
  "cardinality_distribution": <distribution descriptor object>,
  "percent_missing": <percentage value>,
  "percent_nulls": <percentage value>,
  "dimensions": [<list of dimensions nested within the object>]
}
```

Where:
- <i>name</i> is the name of the object
- <i>cardinality</i> indicates the number of unique values for this dimension (zero for unconstrained cardinality)
- <i>cardinality_distribution</i> skews the cardinality selection of the generated objects (optional - omit for unconstrained cardinality)
- <i>percent_missing</i> a value in the range of 0.0 and 100.0 (inclusive) indicating the stochastic frequency for omitting this dimension from records (optional - the default value is 0.0 if omitted)
- <i>percent_nulls</i> a value in the range of 0.0 and 100.0 (inclusive) indicating the stochastic frequency for generating null values (optional - the default value is 0.0 if omitted)
- <i>dimensions</i> is a list of nested dimensions

###### { "type": "list" ...}

list dimensions create lists of dimesions. List dimension specification entries have the following format:

```
{
  "type": "list",
  "name": "<dimension name>",
  "length_distribution": <distribution descriptor object>,
  "selection_distribution": <distribution descriptor object>,
  "elements": [<a list of dimension descriptions>],
  "cardinality": <int value>,
  "cardinality_distribution": <distribution descriptor object>,
  "percent_missing": <percentage value>,
  "percent_nulls": <percentage value>
}
```

Where:
- <i>name</i> is the name of the object
- <i>length_distribution</i> describes the length of the resulting list as a distribution
- <i>selection_distribution</i> informs the generator which elements to select for the list from the elements list
- <i>elements</i> is a list of possible dimensions the generator may use in the generated list
- <i>cardinality</i> indicates the number of unique values for this dimension (zero for unconstrained cardinality)
- <i>cardinality_distribution</i> skews the cardinality selection of the generated lists (optional - omit for unconstrained cardinality)
- <i>percent_missing</i> a value in the range of 0.0 and 100.0 (inclusive) indicating the stochastic frequency for omitting this dimension from records (optional - the default value is 0.0 if omitted)
- <i>percent_nulls</i> a value in the range of 0.0 and 100.0 (inclusive) indicating the stochastic frequency for generating null values (optional - the default value is 0.0 if omitted)


List configuration can seem a bit confusing.
So to clarify, the generator will generate a list that is the length of a sample from the <i>length_distribution</i>.
The types of the elements of the list are selected from the <i>elements</i> list by using an index into the elements list that is determined by sampling from the <i>selection_distribution</i>.
The other field values (e.g., <i>cardinality</i>, <i>percent_nulls</i>, etc.) operate like the other types, but in this case apply to the entire list.


### "interarrival":{}

The _interarrival_ object is a _distribution descriptor object_ that describes the inter-arrival times (in seconds) between records that the driver generates.

One can calculate the mean inter-arrival time by dividing a period of time by the number of records to generate during the time period. For example, 100 records per hour has an inter-arrival time of 36 seconds per record (1 hour * 60 minutes/hour * 60 seconds/minute / 100 records).

See the previous section on _Distribution descriptor objects_ for the syntax.

### "states": []

The _states_ list is a list of state objects.
These state objects describe each of the states in a probabilistic state machine.
The first state in the list is the initial state in the state machine, or in other words, the state that the machine enters initially.

#### State objects

State objects have the following form:

```
{
  "name": <state name>,
  "emitter": <emitter name>,
  "variables": [...]
  "delay": <distribution descriptor object>,
  "transitions": [...]
}
```

Where:
- <i>name</i> is the name of the state
- <i>emitter</i> is the name of the emitter this state will use to generate a record
- <i>variables</i> (optional) a list of dimension objects where the dimension name is used as the variable name
- <i>delay</i> a distribution function indicating how long (in seconds) to remain in the state before transitioning
- <i>transitions</i> is a list of transition objects specifying possible transitions from this state to the next state


##### Transition Objects

Transition objects describe potential state transitions from the current state to the next state.
These objects have the following form:

```
{
  "next": <state name>,
  "probability": <probability value>
}
```

Where:

- <i>next</i> is the name of the next state
- <i>probability</i> is a value greater than zero and less than or equal to one.

Note that the sum of probabilities of all probabilities in the _transitions_ list must add up to one.
Use:

```
"next": "stop",
```

to transition to a terminal state.
