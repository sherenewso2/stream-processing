Ballerina Streaming Guide
=========================

This guide provides an understanding on Ballerina streaming capabilities and you will be learn on building a 
comprehensive streaming usecase with Ballerina Streams.

The following are the sections available in this guide.

* [Overview on Ballerina Streams](#overview)
    * [Stream](#stream)
    * [Forever Statement](#forever-statement)
    * [Query](#query)
        * [Query Projection](#query-projection)
        * [Filter](#filter)
        * [Window](#window)
        * [Aggregation Function](#aggregate-function)
        * [Group By](#group-by)
        * [Having](#having)
        * [Order By](#order-by)
        * [Join](#join)
        * [Pattern](#pattern)
        * [Sequence](#sequence)
        * [Output Rate Limiting](#output-rate-limiting)
* [What you'll build](#what-youll-build)
* [Prerequisites](#prerequisites)
* [Developing queries](#developing-queries)
* [Testing](#testing)
* [Deployment](#deployment)
* [Output](#output)
 
## Overview 

Ballerina Streaming is designed to process event streams in a streaming manner, detect complex event occurrences, 
and notify them in real-time.

Ballerina Streaming supports the following:
 
* Data preprocessing
* Generating alerts based on thresholds
* Calculating aggregations over a short window or a long time period
* Joining multiple data streams
* Correlating data while finding missing and erroneous events
* Detecting temporal event patterns
* Tracking (something over space or time)
* Analyzing trends (rise, fall, turn, tipple bottom)
* And many more ...  


### Stream
A stream is a logical series of events ordered in time. Its schema is defined/constrained via the **record definition**.
A record definition contains a unique name and a set of attributes with specific types and uniquely identifiable names
within the record. All the events that are selected to be received into a specific stream have the same schema
(i.e., have the same attributes in the same order). 

###### Purpose

By defining a schema it unifies common types of events together. This enables them to be processed via queries 
using their defined attributes in a streaming manner.

###### Syntax

The syntax for defining a new stream is as follows.

```sql
type <record name>  {
    <attribute type> <attribute name>;
    <attribute type> <attribute name>;
    <attribute type> <attribute name>;
    ...
};

stream<record name> <stream name>;
```
The following parameters are configured in a stream definition.

| Parameter     | Description |
| ------------- |-------------|
| `stream name`      | The name of the stream created. |
| `attribute name`   | The schema of an record is defined by its attributes with uniquely identifiable attribute names.|
| `attribute type`   | The type of each attribute defined in the record.    |


###### Example
```sql
type Employee {
    string name;
    int age;
    string status;
};

stream<Employee> employeeStream;
```
The above creates a stream named `employeeStream` that constrained by `Employee` type which contain below attributes.

+ `name` of type `string`
+ `age` of type `int` 
+ `status` of type `string` 

### Forever Statement
Stream processing and Complex Event Processing rules can be written in side the forever statement block. Multiple
streaming queries can put together in a single Forever block.

###### Purpose

Each streaming query within forever block is an isolated processing unit that independent to each other.

###### Grammar

Multiple streaming queries can sit together inside a single Forever statement block. Please refer the grammar below.

```antlrv4
foreverStatement
    :   FOREVER LEFT_BRACE  streamingQueryStatement+ RIGHT_BRACE
    ;

streamingQueryStatement
    :   FROM (streamingInput (joinStreamingInput)? | patternClause)
        selectClause?
        orderByClause?
        outputRateLimit?
        streamingAction
    ;
```

###### Example

Query to filter out the teachers who are older than 30 years, wait until three teacher records are collected by the
stream, group the 10 teachers based on their marital status, and calculate the unique marital status count of the
teachers. Once the query is executed, publish the result to the `filteredStatusCountStream` stream.

```sql
    forever {
        from teacherStream where age > 18 window lengthBatch (3)
        select status, count(status) as totalCount
        group by status
        having totalCount > 1
        => (StatusCount [] status) {
                filteredStatusCountStream.publish(status);
        }
    }
```

### Query

Each streaming query can consume one or more streams, process the events in a streaming manner, and then generate an
 output.

###### Purpose

A query enables you to perform complex event processing and stream processing operations by processing incoming events 
one by one in the order they arrive.

###### Syntax

All queries contain an input and an output section. Some also contain a projection section. A simple query with all 
three sections is as follows.

```sql
from <input stream> 
select <attribute name>, <attribute name>, ...
=> (<array type> <parameter name>) {
      ...
      ...
}
```

###### Example

This query consumes events from the `tempStream` stream (that is already defined) and outputs the room temperature and the room number to the `roomTempStream` stream.

```sql
type temperature {
  int deviceID;
  int roomNo;
  float value;  
}

type roomTemperature {
  int roomNo;
  float value;  
}

stream<temperature> tempStream;


from tempStream 
select roomNo, value
=> (roomTemperature[] temperatures) {
    roomTempStream.publish(temperatures);
}
```


#### Query Projection

Streaming queries supports the following for query projections.

<table style="width:100%">
    <tr>
        <th>Action</th>
        <th>Description</th>
    </tr>
    <tr>
        <td>Selecting required objects for projection</td>
        <td>This involves selecting only some of the attributes from the input stream to be inserted into an output stream.
            <br><br>
            E.g., The following query selects only the `roomNo` and `temp` attributes from the `tempStream` stream.
            <pre style="align:left">from tempStream<br>select roomNo, temp<br>=> ( ) { <br/><br/>}</pre>
        </td>
    </tr>
    <tr>
        <td>Selecting all attributes for projection</td>
        <td>Selecting all the attributes in an input stream to be inserted into an output stream. This can be done by using asterisk ( * ) or by omitting the `select` statement.
            <br><br>
            E.g., Both the following queries select all the attributes in the `tempStream` stream.
            <pre>from tempStream<br>select *<br>=> ( ) { <br/><br/>}</pre>
            or
            <pre>from tempStream<br>=> ( ) { <br/><br/>}</pre>
        </td>
    </tr>
    <tr>
        <td>Renaming attributes</td>
        <td>This selects attributes from the input streams and inserts them into the output stream with different names.
            <br><br>
            E.g., This query renames `roomNo` to `roomNumber` and `temp` to `temperature`.
            <pre>from tempStream <br>select roomNo as roomNumber, temp as temperature<br>=> ( ) { <br/><br/>}</pre>
        </td>
    </tr>
    <tr>
        <td>Introducing the constant value</td>
        <td>This adds constant values by assigning it to an attribute using `as`.
            <br></br>
            E.g., This query specifies 'C' to be used as the constant value for `scale` attribute. 
            <pre>from tempStream<br>select roomNo, temp, 'C' as scale<br>=> ( ) { <br/><br/>}</pre>
        </td>
    </tr>
    <tr>
        <td>Using mathematical and logical expressions</td>
        <td>This uses attributes with mathematical and logical expressions in the precedence order given below, and assigns them to the output attribute using `as`.
            <br><br>
            <b>Operator precedence</b><br>
            <table style="width:100%">
                <tr>
                    <th>Operator</th>
                    <th>Distribution</th>
                    <th>Example</th>
                </tr>
                <tr>
                    <td>
                        ()
                    </td>
                    <td>
                        Scope
                    </td>
                    <td>
                        <pre>(cost + tax) * 0.05</pre>
                    </td>
                </tr>
                <tr>
                    <td>
                         == NULL
                    </td>
                    <td>
                        Null check
                    </td>
                    <td>
                        <pre>deviceID == null</pre>
                    </td>
                </tr>
                <tr>
                    <td>
                        !
                    </td>
                    <td>
                        Logical NOT
                    </td>
                    <td>
                        <pre>! (price > 10)</pre>
                    </td>
                </tr>
                <tr>
                    <td>
                         *   /   %  
                    </td>
                    <td>
                        Multiplication, division, modulo
                    </td>
                    <td>
                        <pre>temp * 9/5 + 32</pre>
                    </td>
                </tr>
                <tr>
                    <td>
                        +   -  
                    </td>
                    <td>
                        Addition, substraction
                    </td>
                    <td>
                        <pre>temp * 9/5 - 32</pre>
                    </td>
                </tr>
                <tr>
                    <td>
                        <   <=   >   >=
                    </td>
                    <td>
                        Comparators: less-than, greater-than-equal, greater-than, less-than-equal
                    </td>
                    <td>
                        <pre>totalCost >= price * quantity</pre>
                    </td>
                </tr>
                <tr>
                    <td>
                        ==   !=  
                    </td>
                    <td>
                        Comparisons: equal, not equal
                    </td>
                    <td>
                        <pre>totalCost !=  price * quantity</pre>
                    </td>
                </tr>
                <tr>
                    <td>
                        &&
                    </td>
                    <td>
                        Logical AND
                    </td>
                    <td>
                        <pre>temp < 40 && (humidity < 40 or humidity >= 60)</pre>
                    </td>
                </tr>
                <tr>
                    <td>
                        ||
                    </td>
                    <td>
                        Logical OR
                    </td>
                    <td>
                        <pre>temp < 40 || (humidity < 40 && humidity >= 60)</pre>
                    </td>
                </tr>
            </table>
            E.g., Converting Celsius to Fahrenheit and identifying rooms with room number between 10 and 15 as server rooms.
            <pre>from tempStream<br>select roomNo, temp * 9/5 + 32 as temp, 'F' as scale, roomNo > 10 && roomNo < 15 as isServerRoom<br>=> (RoomFahrenheit [] events ) { <br/><br/>}</pre>
    </tr>
    
</table>

#### Filter

Filters are included in queries to filter information from input streams based on a specified condition.

###### Purpose

A filter allows you to separate events that match a specific condition as the output, or for further processing.

###### Syntax

Filter conditions should be defined with 'where' keyword next to the input stream name as shown below.

```sql
from <input stream> where <filter condition>
select <attribute name>, <attribute name>, ...
=> ( ) {

}
```

###### Example

This query filters all server rooms of which the room number is within the range of 100-210, and having temperature greater than 40 degrees
from the `tempStream` stream, and inserts the results into the `highTempStream` stream.

```sql
from tempStream where (roomNo >= 100 && roomNo < 210) && temp > 40
select roomNo, temp
=> (RoomTemperature [] value) {
    highTempStream.publish(values)
}
```


#### Window

Windows allow you to capture a subset of events based on a specific criterion from an input stream for calculation.
Each input stream can only have a maximum of one window.

###### Purpose

To create subsets of events within a stream based on time duration, number of events, etc for processing.
A window can operate in a sliding or tumbling (batch) manner.

###### Syntax

The `window` prefix should be inserted next to the relevant stream in order to use a window.

```sql
from <input stream> window <window name>(<parameter>, <parameter>, ... )
select <attribute name>, <attribute name>, ...
=> ( ) {

}
```
Note : Filter condition can be applied both before and/or after the window

###### Example

If you want to identify the maximum temperature out of the last 10 events, you need to define a `length` window of 10 events.
 This window operates in a sliding mode where the following 3 subsets are calculated when a list of 12 events are received in a sequential order.

|Subset|Event Range|
|------|-----------|
| 1 | 1-10 |
| 2 | 2-11 |
|3| 3-12 |

The following query finds the maximum temperature out of **last 10 events** from the `tempStream` stream,
and inserts the results into the `maxTempStream` stream.

```sql
from tempStream window length(10)
select max(temp) as maxTemp
=> ( ) {

}
```

If you define the maximum temperature reading out of every 10 events, you need to define a `lengthBatch` window of 10 events.
This window operates as a batch/tumbling mode where the following 3 subsets are calculated when a list of 30 events are received in a sequential order.

|Subset|Event Range|
|------|-----------|
| 1    | 1-10      |
| 2    | 11-20     |
| 3    | 21-30     |

The following query finds the maximum temperature out of **every 10 events** from the `tempStream` stream,
and inserts the results into the `maxTempStream` stream.

```sql
from tempStream window lengthBatch(10)
select max(temp) as maxTemp
=> ( ) {

}
```

Note : Similar operations can be done based on time via `time` windows and `timeBatch` windows and for others.
    Code segments such as `window time(10000)` considers events that arrive during the last 10 seconds in a sliding manner, and the `window timeBatch(2000)` considers events that arrive every 2 seconds in a tumbling manner.

Following are some inbuilt windows shipped with Ballerina Streams.

* time
* timeBatch
* timeLength
* length
* lengthBatch
* sort
* frequent
* lossyFrequent
* cron
* externalTime
* externalTimeBatch


#### Aggregate function

Aggregate functions perform aggregate calculations in the query.
When a window is defined the aggregation is restricted within that window. If no window is provided aggregation is performed from the start.

###### Syntax

```sql
from <input stream> window <window name>(<parameter>, <parameter>, ... )
select <aggregate function>(<parameter>, <parameter>, ... ) as <attribute name>, <attribute2 name>, ...
=> ( ) {

}
```

**Aggregate Parameters**

Aggregate parameters can be attributes, constant values, results of other functions or aggregates, results of mathematical or logical expressions, or time parameters.
Aggregate parameters configured in a query  depends on the aggregate function being called.

###### Example

The following query calculates the average value for the `temp` attribute of the `tempStream` stream. This calculation is done for the last 10 minutes in a sliding manner, and the result is output as `avgTemp` to the `avgTempStream` output stream.

```sql
from tempStream window time(600000)
select avg(temp) as avgTemp, roomNo, deviceID
=> (AvgTemperature [] values) {
    avgTempStream.publish(values);
}
```
Following are some inbuilt aggregation functions shipped with Ballerina, for more aggregation functions, see execution.

* avg
* sum
* max
* min
* count
* distinctCount
* maxForever
* minForever
* stdDev


#### Group By

Group By allows you to group the aggregate based on specified attributes.

###### Syntax

The syntax for the Group By aggregate function is as follows:

```sql
from <input stream> window <window name>(...)
select <aggregate function>( <parameter>, <parameter>, ...) as <attribute1 name>, <attribute2 name>, ...
group by <attribute1 name>, <attribute2 name> ...
=> ( ) {

}
```

###### Example

The following query calculates the average temperature per `roomNo` and `deviceID` combination, for events that arrive at the `tempStream` stream
for a sliding time window of 10 minutes.

```sql
from tempStream window time(600000)
select avg(temp) as avgTemp, roomNo, deviceID
group by roomNo, deviceID
=> (AvgTemperature [] values) {
    avgTempStream.publish(values);
}
```

#### Having

Having allows you to filter events after processing the `select` statement.

###### Purpose

This allows you to filter the aggregation output.

###### Syntax

The syntax for the Having clause is as follows:

```sql
from <input stream> window <window name>( ... )
select <aggregate function>( <parameter>, <parameter>, ...) as <attribute1 name>, <attribute2 name>, ...
group by <attribute1 name>, <attribute2 name> ...
having <condition>
=> ( ) {

}
```

###### Example

The following query calculates the average temperature per room for the last 10 minutes, and alerts if it exceeds 30 degrees.
```sql
from tempStream window time(600000)
select avg(temp) as avgTemp, roomNo
group by roomNo
having avgTemp > 30
=> (Alert [] values) {
    alertStream.publish(values);
}
```

#### Order By

Order By allows you to order the aggregated result in ascending and/or descending order based on specified attributes. By default ordering will be done in
ascending manner. User can use 'descending' keyword to order in descending manner.

###### Syntax

The syntax for the Order By clause is as follows:

```sql
from <input stream> window <window name>( ... )
select <aggregate function>( <parameter>, <parameter>, ...) as <attribute1 name>, <attribute2 name>, ...
group by <attribute1 name>, <attribute2 name> ...
having <condition>
order by <attribute1 name> (ascending | descending)?, <attribute2 name> (<ascend/descend>)?, ...
=> ( ) {

}
```

###### Example

The following query calculates the average temperature per per `roomNo` and `deviceID` combination for every 10 minutes, and generate output events
by ordering them in the ascending order of the room's avgTemp and then by the descending order of roomNo.

```sql
from tempStream window timeBatch(600000)
select avg(temp) as avgTemp, roomNo, deviceID
group by roomNo, deviceID
order by avgTemp, roomNo descending
=> (AvgTemperature [] values) {
    avgTempStream.publish(values);
}
```

#### Join
Joins allow you to get a combined result from two streams in real-time based on a specified condition.

###### Purpose

Streams are stateless. Therefore, in order to join two streams, they need to be connected to a window
so that there is a pool of events that can be used for joining. Joins also accept conditions to join
the appropriate events from each stream.

During the joining process each incoming event of each stream is matched against all the events in the other
stream's window based on the given condition, and the output events are generated for all the matching event pairs.


###### Syntax

The syntax for a join is as follows:

```sql
from <input stream> window <window name>(<parameter>, ... ) {unidirectional} {as <reference>}
         join <input stream> window <window name>(<parameter>,  ... ) {unidirectional} {as <reference>}
    on <join condition>
select <attribute name>, <attribute name>, ...
=> ( ) {

}
```
Here, the `<join condition>` allows you to match the attributes from both the streams.

**Unidirectional join operation**

By default, events arriving at either stream can trigger the joining process. However, if you want to control the
join execution, you can add the `unidirectional` keyword next to a stream in the join definition as depicted in the
syntax in order to enable that stream to trigger the join operation. Here, events arriving at other stream only update the
 window of that stream, and this stream does not trigger the join operation.

Note : The `unidirectional` keyword cannot be applied to both the input streams because the default
behaviour already allows both streams to trigger the join operation.

###### Example

Assuming that the temperature of regulators are updated every minute.
Following is a streaming query that controls the temperature regulators if they are not already `on`
for all the rooms with a room temperature greater than 30 degrees.

```sql
from tempStream where (temp > 30.0) window time(60000) as T
  join regulatorStream where (isOn == false) window length(1) as R
  on T.roomNo == R.roomNo
select T.roomNo, R.deviceID, 'start' as action
=> (RegulatorAction [] values) {
    regulatorActionStream.publish(values);
}
```

**Supported join types**

Following are the supported operations of a join clause.

 *  **Inner join (join)**

    This is the default behaviour of a join operation. `join` is used as the keyword to join both
    the streams. The output is generated only if there is a matching event in both the streams.

 *  **Left outer join**

    The left outer join operation allows you to join two streams to be merged based on a condition.
    `left outer join` is used as the keyword to join both the streams.

    Here, it returns all the events of left stream even if there are no matching events in the right
    stream by having null values for the attributes of the right stream.

     ###### Example

    The following query generates output events for all events from the `stockStream` stream
    regardless of whether a matching symbol exists in the `twitterStream` stream or not.

    <pre>
    from stockStream window time(60000) as S
      left outer join twitterStream window length(1) as T
      on S.symbol== T.symbol
    select S.symbol as symbol, T.tweet, S.price
    => ( ) {

    }   </pre>

 *  **Right outer join**

    This is similar to a left outer join. `Right outer join` is used as the keyword to join both
    the streams. It returns all the events of the right stream even if there are no matching events
    in the left stream.

 *  **Full outer join**

    The full outer join combines the results of left outer join and right outer join. `full outer join` is used as the keyword to join both the streams.
    Here, output event are generated for each incoming event even if there are no matching events in
    the other stream.

    ###### Example

    The following query generates output events for all the incoming events of each stream regardless of whether there is a
    match for the `symbol` attribute in the other stream or not.

    <pre>
    from stockStream window time(60000) as S
      full outer join twitterStream window length(1) as T
      on S.symbol== T.symbol
    select S.symbol as symbol, T.tweet, S.price
    => ( ) {

    }    </pre>


#### Pattern

This is a state machine implementation that allows you to detect patterns in the events that arrive over time.
This can correlate events within a single stream or between multiple streams.

###### Purpose

Patterns allow you to identify trends in events over a time period.

###### Syntax

The following is the syntax for a pattern query:

```sql
from (every)? <event reference>=<input stream> where <filter condition> followed by
    (every)? <event reference>=<input stream where <filter condition> followed by
    ...
    (within <time gap>)?
select <event reference>.<attribute name>, <event reference>.<attribute name>, ...
=> ( ) {

}
```
| Items| Description |
|-------------------|-------------|
| `followed by` | This is used to indicate an event that should be following another event. The subsequent event does not necessarily have to occur immediately after the preceding event. The condition to be met by the preceding event should be added before the sign, and the condition to be met by the subsequent event should be added after the sign. |
| `<event reference>` | This allows you to add a reference to the the matching event so that it can be accessed later for further processing. |
| `(within <time gap>)?` | The `within` clause is optional. It defines the time duration within which all the matching events should occur. |
| `every` | `every` is an optional keyword. This defines whether the event matching should be triggered for every event arrival in the specified stream with the matching condition. <br/> When this keyword is not used, the matching is carried out only once. |

Ballerina Streams also supports pattern matching with counting events and matching events in a logical order such
as (`&&`, `||`, and `!`). These are described in detail further below in this guide.

###### Example

This query sends an alert if the temperature of a room increases by 5 degrees within 10 min.

```sql
from every( e1 = tempStream ) followed by e2 = tempStream where (e1.roomNo == roomNo && (e1.temp + 5) <= temp)
    within 10 minute
select e1.roomNo, e1.temp as initialTemp, e2.temp as finalTemp
=> (Alert [] alerts) {
    alertStream.publish(alerts);
}
```

Here, the matching process begins for each event in the `tempStream` stream (because `every` is used with `e1=tempStream`),
and if  another event arrives within 10 minutes with a value for the `temp` attribute that is greater than or equal to `e1.temp + 5`
of the event e1, an output is generated via the `alertStream`.

##### Counting Pattern

Counting patterns allow you to match multiple events that may have been received for the same matching condition.
The number of events matched per condition can be limited via condition postfixes.

###### Syntax

Each matching condition can contain a collection of events with the minimum and maximum number of events to be matched as shown in the syntax below.

```sql
from (every)? <event reference>=<input stream> where <filter condition> ([<min count> .. <max count>])? followed by
    ...
    (within <time gap>)?
select <event reference>([event index])?.<attribute name>, ...
=> ( ) {

}
```

|Postfix|Description|Example
---------|---------|---------
|`[n1..n2]`|This matches `n1` to `n2` events (including `n1` and not more than `n2`).| `[1..4]` matches 1 to 4 events.
|`[n..]`|This matches `n` or more events (including `n`).|`[2..]` matches 2 or more events.
|`[..n]`|This matches up to `n` events (excluding `n`).|`[..5]` matches up to 5 events.
|`[n]`|This matches exactly `n` events.|`[5]` matches exactly 5 events.

Specific occurrences of the event in a collection can be retrieved by using an event index with its reference.
Square brackets can be used to indicate the event index where `1` can be used as the index of the first event and `last` can be used as the index
 for the `last` available event in the event collection. If you provide an index greater then the last event index,
 the system returns `null`. The following are some valid examples.

+ `e1[3]` refers to the 3rd event.
+ `e1[last]` refers to the last event.
+ `e1[last - 1]` refers to the event before the last event.

###### Example

The following streaming query calculates the temperature difference between two regulator events.

```sql
type Temperature {
    int deviceID,
    int roomNo,
    float temp
};

type Regulator {
    int deviceID,
    int roomNo,
    float tempSet,
    boolean isOn
};

stream<Temperature> tempStream;
stream<Regulator> regulatorStream;

from every( e1=regulatorStream) followed by e2=tempStream where (e1.roomNo==roomNo) [1..] followed by e3=regulatorStream where (e1.roomNo==roomNo)
select e1.roomNo, e2[0].temp - e2[last].temp as tempDiff
=> (TemperatureDiff [] values) {
    tempDiffStream.publish(values);
}
```

##### Logical Patterns

Logical patterns match events that arrive in temporal order and correlate them with logical relationships such as `&&`,
`||` and `!`.

###### Syntax

```sql
from (every)? (!)? <event reference>=<input stream> where <filter condition>
          ((&& | ||) <event reference>=<input stream> where <filter condition>)? (within <time gap>)? followed by
    ...
select <event reference>([event index])?.<attribute name>, ...
=> ( ) {

}
```

Keywords such as `&&`, `||`, or `!` can be used to illustrate the logical relationship.

Key Word|Description
---------|---------
`&&`|This allows both conditions of `&&` to be matched by two events in any order.|
<code>&#124;&#124;</code>|The state succeeds if either condition of <code>&#124;&#124;</code> is satisfied. Here the event reference of the other condition is `null`.|
`! <condition1> && <condition2>`| When `!` is included with `&&`, it identifies the events that match <condition2> arriving before any event that match <condition1>.|
`! <condition> for <time period>`| When `!` is included with `for`, it allows you to identify a situation where no event that matches `<condition1>` arrives during the specified `<time period>`.  e.g.,`from ! temperatureStream where (temp > 60) for 5 second`.|

Here the `!` pattern can be followed by either an `&&` clause or the effective period of `!` can be concluded after a given `<time period>`. Further in Ballerina Streams more than two streams cannot be matched with logical conditions using `&&`, `||`, or `!` clauses at this point.

###### Example

Following streaming query, sends the `stop` control action to the regulator when the key is removed from the hotel room.
```sql

type RegulatorState {
    int deviceID,
    int roomNo,
    float tempSet,
    string action
};

type RoomKey {
    int deviceID,
    int roomNo,
    string action
};

stream<RegulatorState> regulatorStateChangeStream;
stream<RoomKey> roomKeyStream;

from every( e1=regulatorStateChangeStream where (action == 'on')) followed by
      e2=roomKeyStream where (e1.roomNo == roomNo && action == 'removed') || e3=regulatorStateChangeStream where (e1.roomNo == roomNo && action == 'off')
select e1.roomNo, e2 == null ? "none" : "stop" as action
having action != 'none'
=> (RegulatorAction [] output) {
    regulatorActionStream.publish(output);
}
```

This streaming query generates an alert if we have switch off the regulator before the temperature reaches 12 degrees.

```sql

type RegulatorState {
    int deviceID,
    int roomNo,
    float tempSet,
    string action
};

type Temperature {
    int deviceID,
    int roomNo,
    float temp
};

stream<RegulatorState> regulatorStateChangeStream;
stream<Temperature> tempStream;

from e1=regulatorStateChangeStream where (action == 'start') followed by  !tempStream where (e1.roomNo == roomNo && temp < 12) && e2=regulatorStateChangeStream where (action == 'off')
select e1.roomNo as roomNo
=> (Alert [] alerts) {
    alertStream.publish(alerts);
}
```

This streaming query generates an alert if the temperature does not reduce to 12 degrees within 5 minutes of switching on the regulator.

```sql

type RegulatorState {
    int deviceID,
    int roomNo,
    float tempSet,
    string action
};

type Temperature {
    int deviceID,
    int roomNo,
    float temp
};

stream<RegulatorState> regulatorStateChangeStream;
stream<Temperature> tempStream;

from e1=regulatorStateChangeStream where (action == 'start') followed by !tempStream where (e1.roomNo == roomNo && temp < 12) for '5 minute'
select e1.roomNo as roomNo
=> (Alert [] alerts) {
    alertStream.publish(alerts);
}
```


#### Sequence

Sequence is a state machine implementation that allows you to detect the sequence of event occurrences over time.
Here **all matching events need to arrive consecutively** to match the sequence condition, and there cannot be any non-matching events arriving within a matching sequence of events.
This can correlate events within a single stream or between multiple streams.

###### Purpose

This allows you to detect a specified event sequence over a specified time period.

###### Syntax

The syntax for a sequence query is as follows:

```sql
from (every)? <event reference>=<input stream> where <filter condition> ,
    <event reference>=<input stream where <filter condition> ,
    ...
    (within <time gap>)?
select <event reference>.<attribute name>, <event reference>.<attribute name>, ...
=> ( ) {

}
```

| Items | Description |
|-------------------|-------------|
| `,` | This represents the immediate next event i.e., when an event that matches the first condition arrives, the event that arrives immediately after it should match the second condition. |
| `<event reference>` | This allows you to add a reference to the the matching event so that it can be accessed later for further processing. |
| `(within <time gap>)?` | The `within` clause is optional. It defines the time duration within which all the matching events should occur. |
| `every` | `every` is an optional keyword. This defines whether the matching event should be triggered for every event that arrives at the specified stream with the matching condition. <br/> When this keyword is not used, the matching is carried out only once. |


###### Example

This query generates an alert if the increase in the temperature between two consecutive temperature events exceeds one degree.

```sql
from every e1=tempStream, e2=tempStream where (e1.temp + 1 < temp)
select e1.temp as initialTemp, e2.temp as finalTemp
=> (Alert [] alerts) {
    alertStream.publish(alerts);
}
```

##### Counting Sequence

Counting sequences allow you to match multiple events for the same matching condition.
The number of events matched per condition can be limited via condition postfixes such as **Counting Patterns**, or by using the
`[0..]`, `[1..]`, and `[0..1]` operators.

The matching events can also be retrieved using event indexes, similar to how it is done in **Counting Patterns**.

###### Syntax

Each matching condition in a sequence can contain a collection of events as shown below.

```sql
from (every)? <event reference>=<input stream> where <filter condition> ([0..]|[1..]|[0..1])?,
    <event reference>=<input stream where <filter condition>([0..]|[1..]|[0..1])?,
    ...
    (within <time gap>)?
select <event reference>.<attribute name>, <event reference>.<attribute name>, ...
=> ( ) {

}
```

|Postfix symbol|Required/Optional |Description|
|---------|---------|---------|
| `[1..]` | Optional |This matches **one or more** events to the given condition. |
| `[0..]` | Optional |This matches **zero or more** events to the given condition. |
| `[0..1]` | Optional |This matches **zero or one** events to the given condition. |


###### Example

This streaming query identifies temperature peeks.

```sql

type Temperature {
    int deviceID,
    int roomNo,
    float temp
};

stream<Temperature> tempStream;

from every e1=tempStream, e2=tempStream where (e1.temp <= temp)[1..], e3=tempStream where (e2[last].temp > temp)
select e1.temp as initialTemp, e2[last].temp as peakTemp
=>(PeekTemperature [] values) {
    peekTempStream.publish(values);
}
```

##### Logical Sequence

Logical sequences identify logical relationships using `&&`, `||` and `!` on consecutively arriving events.

###### Syntax
The syntax for a logical sequence is as follows:

```sql
from (every)? (!)? <event reference>=<input stream> where <filter condition>
          ((&& | ||) <event reference>=<input stream> where <filter condition>)? (within <time gap>)?,
    ...
select <event reference>([event index])?.<attribute name>, ...
=> ( ) {

}
```

Keywords such as `&&`, `||`, or `!` can be used to illustrate the logical relationship, similar to how it is done in **Logical Patterns**.

###### Example

This streaming query notifies the state when a regulator event is immediately followed by both temperature and humidity events.

```sql

type Temperature {
    int deviceID,
    float temp
};

type Humidity {
    int deviceID,
    float humid
};

type Regulator {
    int deviceID,
    boolean isOn
};

stream<Temperature> tempStream;
stream<Humidity> humidStream;
stream<Regulator> regulatorStream;

from every e1=regulatorStream, e2=tempStream && e3=humidStream
select e2.temp, e3.humid
=> (Notification [] notifications) {
    stateNotificationStream.publish(notifications);
}
```


#### Output rate limiting

Output rate limiting allows queries to output events periodically based on a specified condition.

###### Purpose

This allows you to limit the output to avoid overloading the subsequent executions, and to remove unnecessary
information.

###### Syntax

The syntax of an output rate limiting configuration is as follows:

```sql
from <input stream> ...
select <attribute name>, <attribute name>, ...
output <rate limiting configuration>
=> ( ) {

}
```
Ballerina Streams supports three types of output rate limiting configurations as explained in the following table:

Rate limiting configuration|Syntax| Description
---------|---------|--------
Based on time | `<output event> every <time interval>` | This outputs `<output event>` every `<time interval>` time interval.
Based on number of events | `<output event> every <event interval> events` | This outputs `<output event>` for every `<event interval>` number of events.
Snapshot based output | `snapshot every <time interval>`| This outputs all events in the window (or the last event if no window is defined in the query) for every given `<time interval>` time interval.

Here the `<output event>` specifies the event(s) that should be returned as the output of the query.
The possible values are as follows:
* `first` : Only the first event processed by the query during the specified time interval/sliding window is emitted.
* `last` : Only the last event processed by the query during the specified time interval/sliding window is emitted.
* `all` : All the events processed by the query during the specified time interval/sliding window are emitted. **When no `<output event>` is defined, `all` is used by default.**

###### Examples

+ Returning events based on the number of events

    Here, events are emitted every time the specified number of events arrive. You can also specify whether to emit only the first event/last event, or all the events out of the events that arrived.

    In this example, the last temperature per sensor is emitted for every 10 events.

    <pre>
    from tempStream
    select temp, deviceID
    group by deviceID
    output last every 10 events
    => (LowRateTemperature [] values) {

    }    </pre>

+ Returning events based on time

    Here events are emitted for every predefined time interval. You can also specify whether to emit only the first event, last event, or all events out of the events that arrived during the specified time interval.

    In this example, emits all temperature events every 10 seconds

    <pre>
    from tempStream
    output every 10 second
    => (LowRateTemperature [] values) {

    }    </pre>

+ Returning a periodic snapshot of events

    This method works best with windows. If the input stream is not connected to a window, only the last current event for each predefined time interval is emitted.

    This query emits a snapshot of the events in a time window of 5 seconds every 1 second.

    <pre>
    from tempStream window time(5000)
    output snapshot every 1 second
    => (SnapshotTemperature [] values) {

    }    </pre>


## What you'll build

For better understand let's take a real world usecase and implement that using Ballerina streaming features.

Let's assume, that you are a API developer and you have published few APIs to the API store. There are subscribers who
are subscribed to those APIs as well. 

Here, we are focusing on below scenario where we have a order management service which allow to add order. At this 
situation, you wanted to build an alert generation mechanism which send you an alert in below conditions. 

- No of API/Service requests is greater than 10 in 10 seconds from same IP 

![Stream Processing](images/Streaming.svg "Stream Processing")


## Prerequisites
 
- JDK 1.8 or later
- [Ballerina Distribution](https://github.com/ballerina-lang/ballerina/blob/master/docs/quick-tour.md)
- A Text Editor or an IDE 

### Optional requirements
- Ballerina IDE plugins ([IntelliJ IDEA](https://plugins.jetbrains.com/plugin/9520-ballerina), [VSCode](https://marketplace.visualstudio.com/items?itemName=WSO2.Ballerina), [Atom](https://atom.io/packages/language-ballerina))
- [Docker](https://docs.docker.com/engine/installation/)


## Developing queries

> If you want to skip the basics, you can download the git repo and directly move to "Testing" section by skipping 
"Developing" section.

### Create the project structure

Ballerina is a complete programming language that can have any custom project structure that you wish. Although the 
language allows you to have any package structure, use the following package structure for this project to follow 
this guide.

```
streaming-service
  └── api-alerting
      ├── order_mgt_service.bal
      ├── api_alert.bal
      └── test
          └── order_mgt_service_test.bal          
```
You can create the above Ballerina project using Ballerina project initializing toolkit.

- First, create a new directory in your local machine as `streaming-service` and navigate to the directory using terminal. 
- Then enter the following inputs to the Ballerina project initializing toolkit.
```bash
streaming-service$ ballerina init -i
Create Ballerina.toml [yes/y, no/n]: (y) y
Organization name: (mohan) api-alerting
Version: (0.0.1) 
Ballerina source [service/s, main/m]: (s) s
Package for the service : (no package) api-alerting
Ballerina source [service/s, main/m, finish/f]: (f) f

Ballerina project initialized
```

- Once you initialize your Ballerina project, you can change/add the names of the file to match with our guide project file 
names.

### Implement the streaming queries

- Let's write streaming query as mentioned below. Forever statement contains the streaming queries that relevant for this
usecase/scenario.

##### api_alert.bal
```ballerina

import ballerina/io;

type ClientRequest {
    string host;
};

type RequestCount {
    string host;
    int count;
};

stream<ClientRequest> requestStream;

function initRealtimeRequestCounter () {

    stream<RequestCount> requestCountStream;

    //Whenever the `requestCountStream` stream receives an event from the streaming rules defined in the `forever` block,
    //the `printRequestCount` function is invoked.
    requestCountStream.subscribe(printRequestCount);

    //Gather all the events that are coming to requestStream for ten seconds, group them by the host, count the number
    //of requests per host, and check if the count is more than ten. If yes, publish the output (host and the count) to
    //the `requestCountStream` stream as an alert. This `forever` block is executed once, when initializing the service.
    // The processing happens asynchronously each time the `requestStream` receives an event.
    forever {
        from requestStream
        window time(10000)
        select host, count(host) as count 
        group by host 
        having count > 10
        => (RequestCount [] counts) {
                //The 'counts' is the output of the streaming rules and is published to the `requestCountStream`.
                //The `select` clause should match the structure of the 'RequestCount' struct.
                requestCountStream.publish(counts);
        }
    }
}

// Define the `printRequestCount` function.
function printRequestCount (RequestCount reqCount) {
    io:println("ALERT!! : Received more than 10 requests from the host within 10 seconds: " + reqCount.host);
}

```

- We can get started with a Ballerina service; 'order_mgt_service', which is the RESTful service that serves the order 
management request. order_mgt_service can have multiple resources and each resource is dedicated for a specific order 
management functionality. 

- You can add the content to your Ballerina service as shown below. In that code segment you can find the implementation 
of the service 'order_mgt_service'.
 
For each order management operation, there is a dedicated resource and inside each resource we can implement the 
order management operation logic. Here we are using the order_mgt_service.bal file that used in restful-service guide 
to explain how streaming queries can be used with some modification which calls a function to send an event to the 
stream.

##### order_mgt_service.bal
```ballerina

import ballerina/http;
import ballerina/mime;

function sendRequestEventToStream (string hostName) {
        ClientRequest clientRequest = {host : hostName};
        requestStream.publish(clientRequest);
}

endpoint http:Listener listener {
    port:9090
};

// Order management is done using an in memory map.
// Add some sample orders to 'orderMap' at startup.
map<json> ordersMap;

@Description {value:"RESTful service."}
@http:ServiceConfig {basePath:"/ordermgt"}
service<http:Service> order_mgt bind listener {

    // Invoke function to initialise streming queries
    future ftr = start initRealtimeRequestCounter();

    @Description {value:"Resource that handles the HTTP POST requests that are directed
     to the path '/orders' to create a new Order."}
    @http:ResourceConfig {
        methods:["POST"],
        path:"/order"
    }
    addOrder(endpoint client, http:Request req) {

	string hostName = untaint req.getHeader("Host");
	sendRequestEventToStream(hostName);	

        json orderReq = check req.getJsonPayload();
        string orderId = orderReq.Order.ID.toString();
        ordersMap[orderId] = orderReq;

        // Create response message.
        json payload = {status:"Order Created.", orderId:orderId};
        http:Response response;
        response.setJsonPayload(payload);

        // Set 201 Created status code in the response message.
        response.statusCode = 201;
        // Set 'Location' header in the response message.
        // This can be used by the client to locate the newly added order.
        response.setHeader("Location", "http://localhost:9090/ordermgt/order/" + orderId);

        // Send response to the client.
        _ = client -> respond(response);
    }
}

```

- With that we've completed the development of the order_mgt_service and api_alert implementation. 

### Customize the streaming queries to send email alerts

In Above implementation, we simply generate a log to the stdout. An extended version of the above implementation would
be, sending the alert as an email. Following shows you how to configure the gmail connector to send email as alerts.

- Add following code fragment to api_alert.bal as a global variable (in the same scope the ClientRequest and RequestCount types are defined).
For more information on how ballerina gmail connector is configured, please refer [here](https://github.com/wso2-ballerina/package-gmail/blob/master/Readme.md)
You have to replace "access-token", "client-id", "client-secret", "refresh-token" with your OAuth credentials.
For more information on Google OAuth 2.0 applications, please refer [here](https://developers.google.com/identity/protocols/OAuth2)

```ballerina
endpoint gmail:Client gMailEP {
    clientConfig:{
        auth:{
            accessToken:"access-token",
            clientId:"client-id",
            clientSecret:"client-secret",
            refreshToken:"refresh-token"
        }
    }
};
```

- Replace the function body of 'printRequestCount' with the following code fragment. Then instead of printing a log,
the program will send an email alert to the respective receipient.
you have to replace the recipient@mail.com and sender@mail.com with a correct reciepient and sender email addresses.

```ballerina
gmail:MessageRequest messageRequest;
messageRequest.recipient = "recipient@mail.com";
messageRequest.sender = "sender@mail.com";
messageRequest.subject = "Too many orders!!";
messageRequest.messageBody = "Received more than 10 requests from the host within 10 seconds: " + reqCount.host;
//Set the content type of the mail as TEXT_PLAIN or TEXT_HTML.
messageRequest.contentType = gmail:TEXT_PLAIN;

//Call the GMail endpoint function sendMessage().
var sendMessageResponse = gMailEP -> sendMessage("me", messageRequest);
match sendMessageResponse {
    (string, string) sendStatus => {
        //For a successful message request, returns message and thread id.
        string messageId;
        string threadId;
        (messageId, threadId) = sendStatus;
        io:println("Sent Message Id : " + messageId);
        io:println("Sent Thread Id : " + threadId);
    }
    gmail:GMailError e => io:println(e); //For unsuccessful attempts, returns GMail Error.
}
```

## Testing

As mentioned in previous steps, we have to invoke above developed order management service to get the alert generated 
from streaming queries. We have to send more than 10 requests from same host with in 10 seconds to get an alert generated.


### Invoking the service 

You can run the service that you developed above, in your local environment. Open your terminal and navigate to 
`<SAMPLE_ROOT_DIRECTORY>/streaming-service` and execute the following command.

```
$ballerina run api-alerting
```
NOTE: You need to have the Ballerina installed in you local machine to run the Ballerina service.  

You can test the functionality of the order management service by sending HTTP request to 'order' operation. 
For example, we have used the curl commands to test each operation of OrderMgtService as follows. 

**Create Order** 
```
curl -v -X POST -d \
'{ "Order": { "ID": "100500", "Name": "XYZ", "Description": "Sample order."}}' \
"http://localhost:9090/ordermgt/order" -H "Content-Type:application/json"

Output :  
< HTTP/1.1 201 Created
< Content-Type: application/json
< Location: http://localhost:9090/ordermgt/order/100500
< Transfer-Encoding: chunked
< Server: wso2-http-transport

{"status":"Order Created.","orderId":"100500"} 
```

## Deployment

Once you are done with the development, you can deploy the service using any of the methods that we listed below. 

### Deploying locally

- As the first step you can build a Ballerina executable archive (.balx) of the service that we developed above, 
using the following command. It points to the directory in which the service we developed above located and it will 
create an executable binary out of that. Navigate to the `<SAMPLE_ROOT>/api-alerting/` folder and run the following command. 

```
$ballerina build api-alerting
```

- Once the api-alerting.balx is created inside the target folder, you can run that with the following command. 

```
$ballerina run target/api-alerting.balx
```

- The successful execution of the service should show us the following output. 
```
$ ballerina run target/api-alerting.balx 

ballerina: deploying service(s) in 'target/api-alerting.balx'
ballerina: started HTTP/WS server connector 0.0.0.0:9090
```

### Deploying on Docker

You can run the service that we developed above as a docker container. As Ballerina platform offers native support for 
running ballerina programs on containers, you just need to put the corresponding docker annotations on your service code. 

### Deploying on Kubernetes

- You can run the service that we developed above, on Kubernetes. The Ballerina language offers native support for 
running a ballerina programs on Kubernetes, with the use of Kubernetes annotations that you can include as part of 
your service code. Also, it will take care of the creation of the docker images. So you don't need to explicitly create 
docker images prior to deploying it on Kubernetes. 

Refer [here](https://github.com/ballerina-guides/restful-service#deployment) for more deployment options.


## Output

You will see a log line as shown below, when you invoke the service by satisfying one of the 
criteria for alert.

```
ALERT!! : Received more than 10 requests from the host within 10 seconds: localhost:9090

```
