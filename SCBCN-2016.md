# SCBCN '16

http://scbcn.github.io/

## Designing in the small: principles and practices

[@carlosble](https://twitter.com/carlosble)

- Subtle details have impact in the code
- In [Codesai](http://www.codesai.com/?lang=en) they do a lot of pair programming, even with more than 2 people. That makes them explain all the programming decisions they take. If you program alone that not happens. Normally they do it half of the day because it's hard.
- least astonishment principle

## Workshop: Understanding the Life of Microservices

Viktor Farcic
http://vfarcic.github.io/devops21/workshop.htm

### Intro

- Containers make the infrastructure management easier
- Automation: need to SSH is a smell
- Services need to be stateless to scale-out
- Service discovery: you can't use config files to reference to other services
- DevOps helps in the ops part of agile. Continuous deployment.

### Hands-on

- docker machine
- docker compose
    - Dockerfile
        - EXPOSE ports
        - ENV vars config (12-factor)
        - CMD (e.g go-demo)
        - HEALTHCHECK
    - docker-compose build to build the alpine image (immutable)
        - docker images
- docker 1.12 goodies
    - docker swarm
        - master, 
    - docker network
        - applied to the cluster
    - docker service create
        - you tell swarm declarative what should we have
- cluster building with swarm etc
    - uses https://linux.die.net/man/1/drill to check instead of dig
    - docker service scale
    - kill one swarm -> automatic failover
    - create another one and join
    - docker service ps util
- exposing ports
    - docker reverse proxy with HAProxy
    - routing mesh, ingress port
- upgrading
    - blue/green: costly with many instances
    - rolling upgrades
- docker for aws private beta
    - template in AWS :o
    - cloudformation stuff
    - nice to see what we should have!!!
- vs k8s:
    - less features but easier to understand
- instrumentation
    - https://github.com/gliderlabs/logspout
- latest swarm was released not a long time ago
- https://platform9.com/blog/compare-kubernetes-vs-docker-swarm/
- docker future
    - he thinks its good to move fast now since there is not lot of adoption

## Workshop: ELM 101

####

- https://guide.elm-lang.org/get_started.html
- https://edmz.org/design/2015/07/29/elm-lang-notes.html
- http://rundis.github.io/blog/2016/elm_maybe.html
- http://package.elm-lang.org/packages/elm-lang/core/4.0.0/Platform-Cmd
- https://github.com/zalando/elm-street-404

####

- Maybe && patern matching
- tuplas vs records
- erlm architecture
- ejemplo / hands on
- ej botón random
    - update tiene que ser una func pura
    - rand no es puro
    - hay que enviar un elm Cmd al runtime de elm para ejecutar el rand y el valor de retorno devolver en el update en un mensaje, ese valor se meterá como mensaje en el update
        - 2 ramas en el pattern matching: get_random y el propio random

### Workshop: event storming

- https://github.com/heynickc/awesome-ddd
- https://www.youtube.com/watch?v=4UZZjyQDgT8

### Railway programming

- https://vimeo.com/97344498