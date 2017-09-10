# CraftConf '15

http://manifesto.softwarecraftsmanship.org

# Microservices at Gilt

- Adrian Trenaman | @adrian_trenaman | gilt.com

## The good

- move from Ruby/Java monolithic to Scala (and other) microservices
- move to AWS

## The bad

- complexity mantaining staging envs for all services:
    - deploy dark canaries to prod: test in prod for some uses
- project ownership:
    - who builds it are responsible for running in prod
- deploy
    - tools around AWS + docker
    - deploy to % of users, if ok 24h, deploy to 100%
- APIs
    - REST-like, apidoc.me
- audit + alert
    - systems guys scared of devs full-autonomy deploying at anytime
        - smart alerts cavellc.github.io
- databases: ???

# Elasticsearch

- Alexander Reelsen @spinscale | elastic.co
- “Elasticsearch - the definitive guide”
- things that we didn’t know already:
    - real-time search
    - analytics & kibana
- distributing data:
    - sharding: earch shard is a Lucene index.
    - replicas:
        - give high availability
        - `index.number_of_replicas`
        - index replicas are used in case of node failures
        - also give higher performance since you can have more than 1 node serving a shard
- distributing search is hard
    - complex: get shards, search top N per shard, reduce, get that from the reduced, return data
    - some internal tricks:
        - caching filters because they don’t contribute to score
        - aggregation: use propabilistic data structures
            - `http://info.prelert.com/blog/q-digest-an-algorithm-for-computing-approximate-quantiles-on-a-collection-of-integers`
            - `https://github.com/tdunning/t-digest`
- hardware:
    - CPU: indexing, searching, highliting
        - map threads to cores
    - I/O: indexing, searchinf, merging
        - use SSD! :-)
    - Memory: aggregation and indices data
        - buy as much as you can, can be used for
            - FS cache too
            - file handles
            - memory locking `bootstrap.mlockall`
            - avoids swapping & OOM killer
    - Network: relocation, backup
- distributed fallacies: whatch them out!
- conclusion: speed is key but there are tradeoffs: query vs index time.
- watch out elasticsearch 2.0 & lucene 5.x

# Netflix

- Jeremy Edberg | @jedberg
- Caching
- Queueing
    - absorve pikes
- Cassandra
    - blablabla:
        - availability over consistency
        - writes over reads
        - open source + data stax support
        - distributed with no SPoF
    - tricks
        - multiple copies in multiple places
        - atlas: netfix monitoring tool
            - alert tuning
- platform
    - AWS > Netflix OSS Platform > App code
    - Prana:
        - https://github.com/Netflix/Prana
        - bridge to the OSS Platform in Java from other langs
- break things:
    - do it many times
    - bring chaos with projects like:
        - chaos monkey: kill instances randomly
        - chaos gorilla: kills datacenters
        - chaos kong: shifts zones traffic
    - in distributed system there are 2 kind of network problems:
        - network unavailable
        - network is slow
            - whats slow?
            - how slow is slow?
            - what depends on?
            - Latency monkey project
    - incident reviews
        - human process, no one to blame
        - what went wrong
        - how could detect it sooner
        - how could you prevent it
        - how could you avoid it in the future

# Concurrency

- Paul Butcher | @paulrabutcher | 7 Concurrency Models in Seven Weeks
- Threads:
    - deadlock
    - live lock
    - lock contention
    - scalability
    - priority inversion
- Memory model
    - `http://en.wikipedia.org/wiki/Memory_model_%28programming%29`
    - `http://en.wikipedia.org/wiki/Java_memory_model`
    - java memory model + 2 threads example = LOL !!!!!!!
- CPU evolution
    - clock speed & transtor density
    - parallellism
        - bit-level
        - instruction-level: pipeliving, branch prediction...
- Testing
    - the problem is data that is
        - shared
            - avoid with message passing:
                - actors
                - CSPs: http://en.wikipedia.org/wiki/Communicating_sequential_processes
            - Alan Key: OO is about message passing
        - mutable
            - avoid it with functional programming
        - or don’t avoid both and use clojure’s approach
- Actor model implementations:
    - Erlang / Elixir
    - Scala + Akka
        - examples: BuggyCash
- Books: they are related to specific technologies mostly
    - “Job concurrency in practice” JVM related
- Golang question: popularizes CSP as alternative to the actor model

# The final causal frontier
- Seans Cribbs | @seancribbs
- “Detection of mutual inconsistency in distributed systems”
- Lamport 
    - Lamport Clocks
    - http://en.wikipedia.org/wiki/Lamport%27s_Distributed_Mutual_Exclusion_Algorithm
- Version vectors
- CRDs: Conflict-Free Replicated Datatypes
    - strong eventual consistency: all replicas that recive the same data have the same state

# Keynote D2

- “Building microservices”

# Using logs to build a solid data structure

- Martin Kleppman | @martinkl | LinkedIn
- dataintensive.net book
- Scenario: multiple systems and databases
    - same info is replicated in different databases depending on the features
        - graphs, inverted indexes, document, relational databases, caches...
    - the app is in charge of writing to all the databases when the user updates the info
        - typically to a relational/document databases, a cache and an inverse index for searches
    - data integrity: you can have race conditions between databases!
        - ddbbs have transactions
        - but what if we have multiple databases?
            - distributed transactions?
            - 2-phase methods?
    - stupid simple solutions are the best
        - keep an ordered sequence of writes
        - append-only
        - persistent
        - solution: a log
- log
    - append only thing
    - they are used everywhere: ddbb log, replication, distributed consensus...
    - logs in ddbb storage engines
        - ddbbs implement b-trees data structures
        - and a write-ahead log before writing to the tree to avoid tree corruption
    - logs in ddbb replication => replication log
    - logs in consensus: raft protocol
    - apache kafka stream partitions are logs
- how to fix the app writes to multiple databases mess?
    - append your writes to a log
        - it creates an order to avoid race conditions
        - provices synchronous writes between databases
    - each ddbb consumer reads from the log and writes to its ddbb
    - earch consumer has its own “pointer” so if it fails it remembers where it was poiting to in the log
    - you can now go back in time!
- Q&A: 
    - some stuff about errors
        - he says Event Sourcing guys have like 2 kinds of logs:
            - one for events not processed generated by the app, e.g UserIsLogging
            - once you write it you generate a UserHasLogged event in the second log
            - then you do some tricky logic in there to know what to repeat in case of failure
    - trust kafka and how to integrate.
        - one way is to write normally to ddbb and then to the log
        - check out http://blog.confluent.io/2015/04/23/bottled-water-real-time-integration-of-postgresql-and-kafka

# Interaction Drive Design

- Sandro Mancuso | @sandromancuso
- software issues
    - packaging/namespacing
        - they should reflect the intention of the app
        - not be layer or framework defined
- didn’t like: destroyed CQRS & DDD :-(

# The hidden dimention of refactoring

- Michael Feathers | @mfeathers
- good content but nothing much, presented his program to analyze dyby code
- i fell asleep lol :-|

# Changing Software Physics to Make Problems Disappear

- Sadache Aldrobi | Play Framework co-creator | @Sadache
- prismic.io CMS presentation

# Saga pattern

- Caitie McCaffrey | twitter | @caitie
- distribution transactions:
    - 2-phase commit pattern
        - http://c2.com/cgi/wiki?TwoPhaseCommit
        - http://the-paper-trail.org/blog/consensus-protocols-two-phase-commit/
        - O(n^2)
        - coordinator is a SPoF
    - google spanner
        - global distributed transaction-ready database
        - custom hardware: atomic clocks & GPS + fiber network
    - sagas
- sagas
    - 1987 paper
    - long lived transactions: collection of small transactions
    - they need to be independent
    - each has a compensating transaction which is like the inverse transaction
      to return to the previous state, more or less like an “undo”
    - in a saga as a unit of work, either:
        - all the transactions are correctly executed
        - or a continous subset of transactions is executed correctly
        - trade-off: atomicity for availability
    - design: you divide a transaction in subtransactions and its compensating ones
        - eg single database: successful one
            - begin saga
            - start book hotel subtransaction
            - end book hotel subtransaction
            - start book car rental subtransaction
            - end book car rental subtransaction
            - start book flight subtransaction
            - end book flight subtransaction
            - end saga
        - eg single database: unsuccessful one
            - begin saga
            - start book hotel subtransaction
            - end book hotel subtransaction
            - start book car rental subtransaction
            - abort saga
            - start compensate car rental subtransaction
            - end compensate car rental subtransaction
            - start compensate book hotel subtransaction
            - end compensate book hotel subtransaction
            - end saga
    - same thing in a distributed system?
        - thing in different servers providing the different booking stuff
- distributed saga
    - same sub-request and its compensation sub-requests
    - same eg: successful one
        - begin saga
        - start book hotel sub-request
        - end book hotel sub-request
        - start book car rental sub-request
        - end book car rental sub-request
        - start book flight sub-request
        - end book flight sub-request
        - end saga
    - but with a saga log (durable & distributed) that records this sub-requests
    - uses a SEC: Saga Execution Component
    - what happens with failures? compensating requests
        - aborted saga response
        - start requests fails
        - start requests timeouts
        - SEC crashes
    - complex drawing with lots of stuff
        - `[ '''''' ] <-> [ ''' ] <-> [ service 1 ]`
        - `[ client ] <-> [ SEC ] <-> [ service 2 ]`
        - `[ '''''' ] <-> [ ''' ] <-> [ service 3 ]`
    - you can have failures in requests compensations !!
- example: Halo backend
    - distributed transactions for game results
- more doc
    - www.cs.cornell.edu/andru/cs711/2002fa/reading/sagas.pdf
    - http://vasters.com/clemensv/2012/09/01/Sagas.aspx
    - https://msdn.microsoft.com/en-us/library/jj591569.aspx

# Techniques and Tools For a Coherent Discussion About Performance in Complex Architectures

- Theo Schlossnagle | @postwait | Circonus
- distributed systems hurt and they will give you headaches
    - some people deploy distributed systems without needed
    - some do because they cryed and noticed that needed them
- performance must matter: 
    - must be made relevant and important
    - is good! cause lots of reason
    - people are happy when they don’t notice performance problems
- consistent terminology to avoid argueing about agreeing :-)
- performance metrics:
    - thoughput vs latency: lower latency -> higher throughput
    - time: users notice ~10 ms changes, +100ms experience is bad
    - connectedness: in microservices architecture what happens between microservices matters a lot
- performance culture:
    - very easy to focus in microoptimitzations
    - focus: small individual wins in stuff with lots of use
    - report and celebrate!
- problem with microservices
    - dapper: google paper research.google.com/pubs/pub36356.html
    - latencies!
