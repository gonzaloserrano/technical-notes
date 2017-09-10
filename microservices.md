# Microservices

## Intro

> **Loosely coupled service oriented architecture with bounded contexts**
- Loosely coupled
  - if 2 services need to be updated at the same time they are TIGHTLY coupled => **distributed monolith**
  - you need to deploy independently!
- bounded contexts
  - > DDD deals with large models by dividing them into different Bounded Contexts and being explicit about their **interrelationships**.
  
As we exit the boundaries of a microservice we enter in the boundaries of non-determinism and the world of distributed systems.
  
### Overview

![](https://i.imgur.com/dYAD54v.png)

It's about +dev, -ops.

Bussiness advantages:

- continuous delivery
- devs take care of:
  - beeing on call
  - resource handling
- advantages of a single microservice vs monolith 
  - less complex (SRP)
  - quicker to develop
  - more quality
  - easier to deploy
  - less fear of change
  - more ownership: autonomy, devops
  - better scaling
  
### Topics

#### intro

> **Code is less complex. Interactions are more complex.**

If we reduce risk of change, we increase the rate of change and we go faster without increasing the risk.

#### isolation

Allows:

  - managing failure without cascading.
  - autonomy to
    - take decisions
    - coordinate
  - mobility
    - move services around in runtime
      - behaviour
      - state
    - location transparency: servers need to be addressable
  
#### failure management

- delays: handled by client timeouts
  - edge service timeouts > others timeouts
  - how much to timeout? netflix and uber pass a time that is reduced in each service chain call
- errors: handled by retries
  - do retry if you have load balancing to another service
- resiliency
  - use timeouts, circuit breakers, and bulk-heads to avoid cascading failure.
- idempotency

#### coupling and cohesion

- sync'ed HTTP/REST introduces coupling
- async gives
  - mobility
  - distribution in time and space

#### latency and data consistency

- strong consistency requires coordination
  - expensive in distributed systems (microservices with upper bounds in
    - latency
    - scalability throughput
    - availability
- CAP theorem
- consensus is extremely costly
- avoid coordination of state, avoid distributed transactions
  - in a single microservices is fine because you have strong consistency there
  - what to do? *it's easier to ask forgiveness than it is to get permission*
    - compensation patterns, e.g sagas

#### discovery

How to know where your services are?
- routing: the services public names are needed for network connection
- versioning

#### security

The more isolation, the more security.

- ops safety: 
  - key management
  - secret management
- authentication
- authoritzation

#### monitoritzation / instrumentation

Having the big picture in a distributed microservice system is hard.

- monitorless: Datadog, AWS CloudWatch
- Vizceral project by Netflix
- health checking

#### testing and debugging

- distributed tracing
- use correlation IDs

#### ops

- automation
- infrastructure orchestration
- logging
- failover
- load balancing
- deployment

### Ecosystem
![](https://i.imgur.com/h8D7aI7.png)

## API Gateway

  
Following [this article]( https://www.infoq.com/articles/seven-uservices-antipatterns) advice, an API Gateway helps with several concerns:
![](https://i.imgur.com/ZHEdTb8.png)

- abstracts internal system architecture
- reduces client roundtrips
- groups logic
- end-user authentication
- throttling
- orchestration
- data transformation
- routing


#### versioning and gRPC

- load balance per version: since we use AWS ELBs to balance, and an ELB maps ports to the same group of machines 1 to 1, how can we have multiple versions in production without using multiple ELBs? Does this even make sense?
  - ALBs don't support TCP, they are layer 7 https://forums.aws.amazon.com/thread.jspa?threadID=237605
  - ELB: 
    - elastic: attached to an autoscaling group
    - load: measures machine load
    - balancing: routes to less-used machine
    - ELBs don't support balancing machine
  - go-gRPC needs one TCP port per instance
    - alt: use a proxy?
      - https://github.com/mwitkow/grpc-proxy/issues/1
      - https://github.com/coreos/etcd/tree/master/proxy/grpcproxy

#### other stuff

- there are many microservices-related concerns that we have not yet decided how to deal with them. They depend also on what kind of product we want to build, e.g which features need to be sync, which async.
- should we do a spike about containers and container orchestration? Will it make it easier than building every microservice by ourselves somewhat manually?
  - kubernetes
  - docker swarm
    - [gonzalo] i have done this workshop http://vfarcic.github.io/devops21/workshop.html

### Refs

#### General
- http://martinfowler.com/articles/microservices.html
- https://www.youtube.com/watch?v=b8TDodu5E0k
  - https://github.com/adrianco/spigo
  - https://github.com/Netflix/vizceral
  - https://github.com/adrianco/go-vizceral
- https://github.com/mfornos/awesome-microservices
- https://www.nginx.com/blog/microservices-at-netflix-architectural-best-practices/
- http://aws-de-media.s3.amazonaws.com/images/AWS_Summit_Berlin_2016/sessions/pushing_the_boundaries_1300_microservices_on_aws.pdf
- http://jonasboner.com/bla-bla-microservices-bla-bla/
- http://martinfowler.com/bliki/BoundedContext.html
- https://speakerdeck.com/felipead/how-to-avoid-building-a-distributed-monolith
- http://www.slideshare.net/datawire/avoid-distributed-monoliths
- http://thenewstack.io/ten-commandments-microservices/
- https://www.oreilly.com/ideas/bla-bla-microservices-bla-bla
- https://www.infoq.com/articles/microservices-practical-tips
- https://www.infoq.com/articles/seven-uservices-antipatterns
- http://microservices.io/patterns/server-side-discovery.html
- [Reactive Microservices Architecture](https://s3-eu-west-1.amazonaws.com/uploads-eu.hipchat.com/67814/859993/ZL3aDVV7S4vX3Ib/Reactive_Microservices_Architecture.pdf)

#### API Gateway / Proxy

- http://www.slideshare.net/mashapeinc/microservices-api-gateways
- Vulcand (used by Typeform as a lib): http://vulcand.github.io/quickstart.html#quick-start
- Zuul (investigated by Wallapop): https://github.com/Netflix/zuul/wiki
- Linkerd (supports gRPC): https://linkerd.io/overview/what-is-linkerd/


#### Ideas / quotes

> Treat servers, particularly those that run customer-facing code, as interchangeable members of a group. They all perform the same functions, so you don’t need to be concerned about them individually. Your only concern is that there are enough of them to produce the amount of work you need, and you can use autoscaling to adjust the numbers up and down. If one stops working, it’s automatically replaced by another one. Avoid “snowflake” systems in which you depend on individual servers to perform specialized functions.
Cockcroft’s analogy is that you want to think of servers like cattle, not pets. If you have a machine in production that performs a specialized function, and you know it by name, and everyone gets sad when it goes down, it’s a pet. Instead you should think of your servers like a herd of cows. What you care about is how many gallons of milk you get. If one day you notice you’re getting less milk than usual, you find out which cows aren’t producing well and replace them.

@todo add conclusions

> These microservice diagrams may look complicated, but looking inside a monolith would be even more confusing because it’s tangled together in ways you can’t even see. The system gets tangled together, like a big mass of spaghetti
With so many services all evolving at different paces and different services rolling out canary releases internally, it can be difficult to recreate environments in a consistent way for either manual or automated testing. When we add in asynchronicity and dynamic message loads, it becomes much harder to test systems built in this style and gain confidence in the set of services that we are about to release into production.

@todo add conclusions

> It is unfortunate that synchronous HTTP is widely considered as the go-to Microservice communication protocol. Its synchronous nature introduces strong coupling between services which makes it a very bad default protocol for inter-service communication.
Instead, communication between Microservices needs to be based on Asynchronous Message-Passing. Having an asynchronous boundary between services is necessary in order to decouple them, and their communication flow:
   in time: for concurrency, and
   in space: for distribution and mobility
    
@todo add conclusions

> If a service only has one single reason to exist, providing a single composable piece of functionality, then business domains and responsibilities are not tangled. Which makes the code and system easier to understand, compose, extend and maintain over time.
However, one Microservice is not of much use, they come in systems.Like humans they act autonomously and therefore need to communicate and collaborate with others to solve problems—and as with humans, it is in collaboration that both the most interesting opportunities and challenging problems arise.Individual Microservices are fairly easy to design and implement—what is hard in Microservices is all the things around them: discovery, coordination, security, replication, data consistency, failover, deployment, and integration with other systems, just to name a few.
    
@todo add conclusions

> The evils of too much coupling between services are far worse than the problems caused by code duplication
The alternative for Christensen is contracts and protocols, services should hide all their implementation details only exposing data contracts and network protocols. Without any dependency on service implementation a consumer can use any technology and language, and evolve at its own pace noticing that this is how Internet works. He notes though that there are legitimate needs for standardization in areas like logging, distributed tracing, routing, etc., but this should be enabled using independent libraries that a consumer can choose whether to use or not.

@todo add conclusions

> Creating multiple, technical, physical layers of services would only cause delivery complexity and runtime inefficiency. We ended up in having wrapper services, orchestration services, business services and data services. These service models served technical concerns. Individual teams formed to manage these layers and ended up having business logic sprawl, no single owner for a capability, lost the efficiency and there was always a blaming game.

@todo add conclusions
