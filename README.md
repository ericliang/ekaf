# ekaf #

An advanced but simple to use, Kafka producer written in Erlang

### Highlights of v0.1 ###
* A minimal implementation of a `Producer` as per 0.8 Kafka wire protocol
* Produce data to a Topic syncronously and asynchronously
* Option to batche messages and customize the concurrency
* Several lazy handling of both metadata requests as well as connection pool creation

ekaf also powers `kafboy`, the webserver based on `Cowboy` capable of publishing to ekaf via simple HTTP endpoints.

Add a feature request at https://github.com/helpshift/ekaf or check the ekaf web server at https://github.com/helpshift/kafboy

##Features##
###Simple API###

Topic is a binary. and the payload can be a list, a binary, a key-value tuple, or a combination of all the above.

    %%------------------------
    %% Start
    %%------------------------
    application:load(ekaf),

    %%%% mandatory. ekaf needs atleast 1 broker to connect.
    %% To eliminate a SPOF, use an IP to a load balancer to a bunch of brokers
    application:set_env(ekaf, {ekaf_bootstrap_broker,{"localhost",9091}}),


    application:start(ekaf),

    Topic = <<"ekaf">>,

    %%------------------------
    %% Send message to a topic
    %%------------------------

    %% sync
    ekaf:produce_sync(Topic, <<"foo">>).

    %% async
    ok = ekaf:produce_async(Topic, jsx:encode(PropList) ).

    %%------------------------
    %% Send events in batches
    %%------------------------

    %% send many messages in 1 packet by sending a list to represent a payload
    {{sent, _OnPartition, _ByPartitionWorker}, _Response }  =
        ekaf:produce_sync(
            Topic,
            [<<"foo">>, {<<"key">>, <<"value">>}, <<"back_to_binary">> ]
        ),

    %%------------------------
    %% Send events in batches
    %%------------------------

    %% sync
    {buffered, _Partition, _BufferSize} =
        ekaf:produce_async_batched(
            Topic,
            [ekaf_utils:itob(X) || X<- lists:seq(1,1000) ]
        ).

    %% async
    {buffered, _Partition, _BufferIndex} =
        ekaf:produce_async_batched(
            Topic,
            [<<"foo">>, {<<"key">>, <<"value">>}, <<"back_to_binary">> ]
        ).

    %%------------------------
    %% Other helpers that are used internally
    %%------------------------
    %% reads ekaf_bootstrap_topics. Will also start their workers
    ekaf:prepare(Topic).
    %% metadata
    %% if you don't want to start workers on app start, make sure to
    %%               first get metadata before any produce/publish
    ekaf:metadata(Topic).
    %% pick a worker, and directly communicate with it
    ekaf:pick(Topic,Callback).
    %% see the tests for a complete API, and `ekaf.erl` and `ekaf_lib` for more

See `test/ekaf_tests.erl` for more

### Tunable Concurrency ###
Each worker represents a connection to a broker + topic + partition combination.
You can decide how many workers to start for each partition

    %% a single topic with 3 partitions will have 15 workers per node
    %% all able to batch writes
    {ekaf,[{ekaf_per_partition_workers, 5},
           {ekaf_per_partition_workers_max, 10}]}.

You can also set different concurrency strategies for different topics

    {ekaf,[{ekaf_per_partition_workers, [
       {<<"topic1">>, 5},              % lazily start a connection pool of 5 workers
       {<<"topic2">>, 1},              % per partition
       {ekaf_max_buffer_size, 1000}    % for remaining topics
    ]}]}.

By lazily starting a connection pool, these workers only are spawned on receipt of the first produce message received by that worker. More on this below.

### Batch writes ###

You can batch async and sync calls until they reach 1000 for a partition, or 5 seconds of inactivity, which ever comes first.

    {ekaf,[
        {ekaf_max_buffer_size, 100},      %% default
        {ekaf_buffer_ttl     , 5000}      %% default
    ]}.


You can also set different batch sizes for different topics.

    {ekaf,[{ekaf_max_buffer_size, [
       {<<"topic1">>, 500},            % topic specific
       {<<"topic2">>, 10000},
       {ekaf_max_buffer_size, 1000}    % for remaining topics
    ]}]}.

You can also change the default buffer flush on inactivity from 1 second. Topic specific is again possible.

    {ekaf,[
        {ekaf_partition_strategy, 1000}  % if a buffer exists after 1 sec on inactivity, send them
    ]}.

### Partition choosing strategy

#### Random Strategy ( Faster, since order not maintained among partition workers )

![ordered_round_robin](/benchmarks/n30000_c100_strategy_random.png)

Will deliver to kafka ordered within the same partition worker, but produced messages can go to any random partition worker. As a result, ordering finally at the kafka broker cannot be ensured.

#### Ordered Round Robin

![ordered_round_robin](/benchmarks/n30000_c100_strategy_sticky_batch.png)

Will attempt to deliver to kafka in the same order than was published by sharding 1000 writes at a time to the same partition worker before switching to another worker. The same partition worker order need not be chosen when run over long durations, but whichever partition worker is picked, its writes will be in order.

### No need for Zookeeper ###
Does not need a connection to Zookeeper for broker info, etc. This adopts the pattern encouraged from the 0.8 protocol

### No linked drivers, No NIF's, No Deps.   ###
Deals with the protcol completely in erlang. Pattern matching FTW, see the blogpost that inspired this project ( also see https://coderwall.com/p/1lyfxg ).

### Optimized for using on a cluster ###
* Works well if embedding into other OTP/rebar style apps ( eg: tested with `kafboy`)
* Only gets metadata for the topic being published. ekaf does not start workers until a produce is called, hence easily horizontally scalable - can be added to a load balancer without worrying about creating holding up valuable connections to a broker on bootup. Queries metadata directly from a `{ekaf_bootstrap_broker,Broker}` during the first produce to a topic, and then caches this data for that topic.
* Extensive use of records for `O(1)` lookup
* By using binary as the preferred format for Topic, etc, - lists are avoided in all places except the `{BrokerHost,_Port}`.

### Concurrency when publishing to multiple topics via process groups ###
Each Topic, will have a pg2 process group, You can pick a random partition worker for a topic via

    ekaf:pick(<<"topic_foo">>, fun(Worker) ->
        case Worker of
            {error, try_again}->
                %% need to get some metadata first for this topic
                %% see how `ekaf:produce_sync/2` does this
            SomeWorkerPid ->
                %% gen_fsm:sync_send_event(SomeWorker, pool_name)
                %% SomeWorker ! info
        end
    end).

    %% pick/1 and pick/2 also exist for synchronously choosing a worker

### State Machines ###
Each worker is a finite state machine powered by OTP's gen_fsm as opposed to gen_server which is more of a client-server model. Which makes it easy to handle connections breaking, and adding more features in the future. In fact every new topic spawns a worker that first starts in a bootstrapping state until metadata is retrieved. This is a blocking call.

Choosing a worker is done by a worker of `ekaf_server` for every topic. It looks at the strategy and decides how often to choose a worker and is used internally by `ekaf:pick`

### An example ekaf config

    {ekaf,[

        % required.
        {ekaf_bootstrap_broker, {"localhost", 9091} },
        % pass the {BrokerHost,Port} of atleast one permanent broker. Ideally should be
        %       the IP of a load balancer so that any broker can be contacted


        % optional.
        {ekaf_bootstrap_topics, [ <<"topic">> ]},
        % will start workers for this topic when ekaf starts
        % a lazy and more recommended approach is to ignore this config

        % optional
        {ekaf_per_partition_workers,100},
        % how big is the connection pool per partition
        % eg: if the topic has 3 partitions, then with this eg: 300 workers will be started


        % optional
        {ekaf_max_buffer_size, [{<<"topic">>,10000},                % for specific topic
                                {ekaf_max_buffer_size,100}]},       % for other topics
        % how many events should the worker wait for before flushing to kafka as a batch


        % optional
        {ekaf_partition_strategy, random},
        % if you are not bothered about the order, use random for speed
        % else the default is ordered_round_robin

    % optional
    {ekaf_callback_flush, {mystats,callback_flush}}
    % can be used for instrumentating how how batches are sent & hygeine

    ]},

### Powering kafboy, a HTTP gateway to ekaf ###

`kafboy` ( https://github.com/helpshift/kafboy ) sits on top of ekaf and is powered by `Cowboy` to send JSON posted to it directly to kafka

    POST /async/topic_name
    POST /batch/async/topic
    POST /safe/batch/async/topic
    etc

# Benchmarks #
Running the test on a 2GB RAM vagrant VM, where the broker was local

    Test_Async_Multi_Batched = fun(N) ->Seq = lists:seq(1,N), N1 = now(),  ekaf:produce_async_batched(<<"ekaf">>, [ ekaf_utils:itob(X) || X <- Seq]), N2 = now(), N/(timer:now_diff(N2,N1)/1000000) end.

    (node@127.0.0.1)28> Test_Async_Multi_Batched(1000).
    202429.14979757086
    (node@127.0.0.1)29> Test_Async_Multi_Batched(10000).
    438981.56277436344
    (node@127.0.0.1)30> Test_Async_Multi_Batched(100000).
    440893.7798705536

#### test multiple calls to the async batch

    Test_Async_Batched = fun(N) ->Seq = lists:seq(1,N), N1 = now(), [ ekaf:produce_async_batched(<<"ekaf">>, ekaf_utils:itob(X)) || X <- Seq], N2 = now(), N/(timer:now_diff(N2,N1)/1000000) end

    (node@127.0.0.1)24> Test_Async_Batched(1000).
    9628.623973347969
    (node@127.0.0.1)25> Test_Async_Batched(10000).
    6209.853298425678
    (node@127.0.0.1)26> Test_Async_Batched(100000).
    6156.06385217173

#### test multiple calls to the async without batch

    Test_Async = fun(N) -> Seq = lists:seq(1,N), N1 = now(), [ ekaf:produce_async(<<"ekaf">>, ekaf_utils:itob(X)) || X <- Seq], N2 = now(), N/(timer:now_diff(N2,N1)/1000000) end.
    (node@127.0.0.1)31>     Test_Async = fun(N) -> Seq = lists:seq(1,N), N1 = now(), [ ekaf:produce_async(<<"ekaf">>, ekaf_utils:itob(X)) || X <- Seq], N2 = now(), N/(timer:now_diff(N2,N1)/1000000) end.
    (node@127.0.0.1)32> Test_Async(1000).
    5030.155783924628
    (node@127.0.0.1)33> Test_Async(10000).
    2771.468068807792
    (node@127.0.0.1)34> Test_Async(100000).
    2427.0491521382896

### Tests ###

The tests assume you have a topic `ekaf`. Create it as instructed on the Kafka Quickstart at `http://kafka.apache.org/08/quickstart.html`

    $ bin/kafka-create-topic.sh --zookeeper localhost:2181 --replica 1 --partition 1 --topic ekaf
ekaf works well with rebar.

    $ rebar get-deps clean compile eunit

    ==> ekaf (eunit)
    Compiled src/ekaf.erl
    Compiled src/ekaf_fsm.erl
    Compiled src/ekaf_picker.erl
    Compiled src/ekaf_protocol.erl
    Compiled src/ekaf_lib.erl
    Compiled src/ekaf_protocol_metadata.erl
    Compiled src/ekaf_server.erl
    Compiled src/ekaf_sup.erl
    Compiled src/ekaf_protocol_produce.erl
    Compiled src/ekaf_utils.erl
    Compiled test/ekaf_tests.erl
    test/ekaf_tests.erl:23:<0.485.0>: t_pick_from_new_pool ( ) = ok
    test/ekaf_tests.erl:25:<0.694.0>: t_request_metadata ( ) = ok
    test/ekaf_tests.erl:27:<0.698.0>: t_request_info ( ) = ok
    test/ekaf_tests.erl:30:<0.702.0>: t_produce_sync_to_topic ( ) = ok
    test/ekaf_tests.erl:32:<0.706.0>: t_produce_sync_multi_to_topic ( ) = ok
    test/ekaf_tests.erl:34:<0.710.0>: t_produce_sync_in_batch_to_topic ( ) = ok
    test/ekaf_tests.erl:36:<0.714.0>: t_produce_sync_multi_in_batch_to_topic ( ) = ok
    test/ekaf_tests.erl:39:<0.718.0>: t_produce_async_to_topic ( ) = ok
    test/ekaf_tests.erl:41:<0.723.0>: t_produce_async_multi_to_topic ( ) = ok
    test/ekaf_tests.erl:43:<0.728.0>: t_produce_async_in_batch_to_topic ( ) = ok
    test/ekaf_tests.erl:45:<0.732.0>: t_produce_async_multi_in_batch_to_topic ( ) = ok
    All 22 tests passed.
    Cover analysis: /data/repos/ekaf/.eunit/index.html

    Code Coverage:
    ekaf                   : 64%
    ekaf_fsm               : 60%
    ekaf_lib               : 64%
    ekaf_picker            : 55%
    ekaf_protocol          : 88%
    ekaf_protocol_metadata : 78%
    ekaf_protocol_produce  : 67%
    ekaf_server            : 36%
    ekaf_sup               : 30%
    ekaf_utils             : 13%

    Total                  : 53%

    =INFO REPORT==== 9-Apr-2014::10:01:42 ===
    application: ekaf
    exited: stopped
    type: temporary

    ![ordered_round_robin](/benchmarks/screenshot-eunit-kafka-consumer.png)

## License

```
Copyright 2014, Helpshift, Inc.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

     http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```

### Goals for v0.2 ###
* Compression when publishing
* Add a feature request at https://github.com/helpshift/ekaf or check the ekaf web server at https://github.com/helpshift/kafboy
