# GoUK '17

Notes from [@gonzaloserrano](https://twitter.com/gonzaloserrano)  
Last year's Ultimate Go workshop notes https://hackmd.io/s/ByS4NKjc  
Last year's talks notes https://hackmd.io/s/B1CAXxXq

## Workshop

Mark Bates - [@markbates](https://twitter.com/markbates)  
author of https://github.com/gobuffalo/buffalo  
slides: http://golanguk.ngrok.io  
files: https://www.dropbox.com/s/c5aecqoivhb5g2n/2017-golanguk-workshop.zip?dl=0

- general review of `error`, `panic()`, `defer`, `recover()`.
- errors:
    - sentinel errors let us know what's happening in our app, e.g `io.EOF` or `sql.ErrNoRows`. They are interesting for control flow. The speaker uses the stdlib ones but does not create them in his code. Does not say the cons.
    - https://github.com/pkg/errors to wrap errors and see tracing data.
- concurrency:
    - `WaitGroup`: don't do `wg.Add(1)` inside the goroutine its doing the `Done()`.
    - example of https://github.com/golang/go/wiki/CommonMistakes#using-goroutines-on-loop-iterator-variables
    - buffered channels: 
        - they can hide a design mistake, so be really careful with them. 
        - what size do you use? Why? 
        - why you don't want to block the sender? 
        - what about backpressure (more writes in than writes out)? You can avoid memory overflow, but what limit you put? Why that limit?
        - they can hide a message loss problem.
    - `select`: be careful of ranging with a `select` with an empty `default` branch, the CPU will suffer.
- testing:
    - https://github.com/smartystreets/goconvey is a cool tool. (Note: I knew about its BDD testing lib which I don't like, but the _reactive_ testing tool is cool).
- dependency management: dep
- context: API, cancelation propagation, trees
- build tools, ldflags
- protobufs & gRPC

## Keynote

Steve Francia - [@spf13](https://twitter.com/spf13)  
slides: http://spf13.com/presentation/stateofthegophernation-aug17/

- contributing to go is easier, see https://blog.golang.org/contributor-workshop

## Writing beautiful packages

Mat Ryer - [@matryer](https://twitter.com/matryer)

- single interface method interfaces allow functions to implement the interface
    - e.g `http.Handler` and `http.HandlerFunc`
- https://rakyll.org/style-packages
- for libs, leave concurrency to the user
- learn from the stdlib
- the pkg name is part of the API, and naming is hard
- take advantadge of zero values and avoid constructors *if you can* - e.g if you have not many fields
- allow injecting `http.Client`
- avoid global state and `init`

## Can you write an OS Kernel in Go?

Achilleas Anagnostopoulos - [@achilleasa](https://github.com/achilleasa)  
slides: https://speakerdeck.com/achilleasa/bare-metal-gophers-can-you-write-an-os-kernel-in-go  
demo code: https://github.com/achilleasa/bare-metal-gophers  
Note: Achilleas wrote https://medium.com/geckoboard-under-the-hood/introducing-prism-9c08e9926755, looks like a great profiling tool with performance diff between changes.

- unikernels and ring 0: avoid kernel / hypervisor overhead
- is Go OK?
    - is GC'ed, but its fast enough for soft RT
- awesome demo: text in framebuffer, image as text, then animation, then a 3D renderer


## Concurrency patterns

Arne Claus from Trivago - [@arnecls](https://twitter.com/arnecls)  
slides: https://speakerdeck.com/arnecls/concurrency-patterns-in-go

- blocking channels
    - channel blocks when no data:
        - no receiver, for unbuffered buffers
        - no sender, for all channels
    - blocking channels are good for synchronizing goroutines
    - blocking can lead to
        - deadlocks
        - scaling problems: adding more blocking goroutines can lead to worse performance
- closing channels
    - closing a channel sends a special _closed_ message to all the readers
    - send after close panics
    - closing twice panics
    - _closed_ makes the reader receive two things: the zero value of the type of the channel, and _false_.
    - the receiver always knows if the channel is closed,  the sender does not.
        - corolary: **always close the channel from the receiving side, not from the sending side!**
- `select`
    - order of cases does not matter
    - a _default_ case exists that's executed if the other cases are blocked
    - to make channels nonblocking, use `time.After` in `select` or `default`
- channels are streams of data; combining streams is powerful. Depending on the _shape_ of the combination there are different patterns:
    - fan-out (1:N): `select` with **writes** to several channels sends to the first non-blocking channel
    - turn-out (N:M): `select` with multiple reads, to `select` with multiple writes.
    - quit channel for cancelation
- channel failures
    - deadlocks
    - memory copying and performance
    - pasing pointers and race conditions
    - caches are about sharing, a cache with channels is not a good idea, use mutex instead?
        - RWLocks can reduce the problem
        - multiple mutexes _will_ cause deadlocks 
- three shades of code
    - blocking: the program can be locked
    - lock free: at least one part of the program makes progress
    - wait free: all parts of the program make progress
- `sync.Atomic` ops are thread-safe because they are based on CPU instructions
- Spin Lock or Spinning CAS: Compare and Swap in a loop. That's how mutex are implemented (?)

## go-swagger

Myles McDonnell - [@mylesmcdonnell](https://twitter.com/mylesmcdonnell)  
slides https://prezi.com/view/wuB2jT1XtDb4IT65S4DS/ 

- https://swagger.io
- https://goswagger.io
- https://github.com/go-swagger/go-swagger
- define RESTful APIs
- goswagger codegen
- go-kit does not support it yet https://github.com/go-kit/kit/issues/185

## embedding

Sean Kelly - [@stabbycutyou](https://twitter.com/stabbycutyou)  
https://github.com/stabbycutyou/embeddingtalk

- inheritance does not exist in Go
    - > embedding is not _better_ than inheritance, is something different to resolve a different problem
    - embedding is for composing interfaces and structs
    - behaviour over lineage
    - no base class - super class relationship
        - "is-a" vs "has-a"
    - method dispatching has some edge cases
- what the spec says:
    - https://golang.org/ref/spec#Struct_types
    - > A field declared with a type but no explicit field name is an anonymous field (colloquially called an embedded field). Such a field type must be specified as a type name T or as a pointer to a non-interface type name \*T, and T itself may not be a pointer type. The unqualified type name acts as the field name.
    - https://golang.org/ref/spec#Interface_types
    - > An interface T may use a (possibly qualified) interface type name E in place of a method specification. This is called embedding interface E in T; it adds all (exported and non-exported) methods of E to the interface T.
    - selectors: https://golang.org/ref/spec#Selectors
        - shallowest depth concept
        - promoted fields: https://golang.org/ref/spec#Struct_types
- examples:
    - emdedding a struct with another with the same field name
        - the field promoted is the one from the embedding not from the embedded struct
            - if you want to access the embedded one you need to do something like `embedding.embedded.field`
    - embedding multiple structs
    - nest embeddings
    - you cannot embed twice the same thing
    - same method and field example https://github.com/StabbyCutyou/embeddingtalk/blob/master/5.multiple/main.go
    - embed interfaces in structs
        - abstract behaviours instead of concrete behaviours
        - he does not say it but it's greate for test doubles where one just use/overwrite some of the methods of the interface
            - warning: explodes at runtime
    - viewmodels to avoid marshaling some model fields https://github.com/StabbyCutyou/embeddingtalk/blob/master/7.marshalling/1.viewmodel/main.go
    - extending generated code
        - embedding generating code makes easy to extend it without modifying the generated code so it can be generated over and over again.
    - promoted methods are only called in the original receiver **[warning]**

## fighting latency & profiling

Filippo Valsorda - [@FiloSottile](https://twitter.com/FiloSottile)  
slides: https://speakerdeck.com/filosottile/you-latency-and-profiling-at-golanguk-2017
#
- definition of fast
    - regex: MBs of data per second
    - API: many clients, _OR_ response time
        - so fast is **throughput** AND **latency**
    - go GC: better latency with every release, probably less throughput too.
- CPU profiling
    - SIGPROF signal + `signal/runtime.go`
- example of pprof for CPU
    - write tmp files to disk
    - profile before optimize!
    - optimizing CPU does not help much (maybe ~15%)
- CPU profiling is for **throughput**
- the tracer (new tool) is for **latency**
    - gathers discrete events, vs samples from the profiler, so its better
    - ways to use: import and pprof
    - has a web UI
    - go tool trace -pprof=TYPE trace.out
        - TYPE = {net, syscall, cpu, mem}
        - syscall for IO wait
- tracing events can be analyzed and new tools can be created to for e.g filter them, like https://github.com/FiloSottile/tracetools/tree/master/cmd/tracefocus 
    - would be great to have more tools!

## contributing to Go

Audrey Lim

- real-life example
- look for `help-wanted` issues in GitHub

## how we built gopherize.me

Mat Ryer - [@matryer](https://twitter.com/matryer)  
Ashley McNamara - [@ashleymcnamara](https://twitter.com/ashleymcnamara)

- looks like a fun project
- here you are mine https://gopherize.me/gopher/f242f064bb638bb1ef52f7d88e6fe97666f79003

## goroutines optimitzation

Guido Patanella

- goroutines are multiplexed into OS threads
- how a goroutine ends up in a CPU processor?

![](https://i.imgur.com/knmeePb.jpg)

- `ps -T | grep <binary-name>`
    - to see how many threads (SPIDs) are run by your go program
- multiple workers example
    - tasks to do VS completed tasks
    - lets put more goroutines: 100, 1000...
    - there is a point where there is no benefit
        - context switching cost
        - map of executions to threads cost
- vertical application scalability
    - requires
        - scheduler
        - orchestation
    - not necessary faster due to overhead 
    - the go runtime gives you that for free
       - native
       - lightweight
- after benchmarking you find the sweet spot of goroutines
- to scale horizontally you need a cluster of servers
    - cluster scheduler
    - workers in each server + handshaking
- other resources 
    - presents a way to limit resources: files, CPU limit (didn't pay much attention on how he tried to do it)
    
## static analysis

Takuya Ueda - [@tenntenn](https://twitter.com/tenntenn)  
slides: https://www.slideshare.net/takuyaueda967/static-analysis-in-go

- reflection to analyze code
- tools (used all of them :-)
    - gofmt / goimports
    - go vet / golint
    - guru
    - gocode
    - errcheck
    - gorename / gomvpkg 
- there are many go subpkgs in the stdlib to do static analysis: ast, build, parser, scanner, token...
- go is very easy to analyze because the language is simple
- flow + steps breakdown
![](https://i.imgur.com/30nLNIY.png)
- examples of all the steps
    - interested content in the slides but too low level for a talk or take notes

## deep learning with go

Chris Benson - [@chrisbenson](https://twitter.com/chrisbenson)

![](https://i.imgur.com/VcJl0r3.jpg)

- deep neural networks to achieven machine learning
- learn from data, not algorithms
- use cases
    - anomaly detection
    - recommenders
    - computer vision, recognition
    - health diagnosis
    - financial analysis
    - ...
    - there are applications for ALL industries
- super growth because of larger datasets and more powerful computers
- neural networks are universal because they approximate to a programming function with at the same time is a universal unit of computation
    - that's why deep learning is so versatile
- Google CEO: next 10 years are about AI 
    - use AI to solve users problems
- AI/ML is not just for data scinetists, putting it to production requires devs
    - you don't need to be an expert to go into the AI/ML space (as before did)
- definitions:
    - gather knowledge from experience, experience to graph of concepts, that are simplified.
    - we like the one from https://mitpress.mit.edu/books/deep-learning
- training a network: generalize predictions using correct results
    - is not memoritzation for results
- how it works: back propagation (the grandfather of all the architectures)
    - data flows through layers of the network![](https://i.imgur.com/kum1Fqh.jpg)
- network architectures: 
    - feed-forward: like the original.
    - convolutional: for vision recognition
    - recurrent: speech/writing recognition
    - generative adversarial (brand new): extends the dataset as you go, you don't need a big dataset
- tensorflow
    - works for go but just for trained networks
        - you have to do the training in python
        - go alternative: https://github.com/sjwhitworth/golearn
        - the tensorflow team needs help in the go side
- more links (not necessary from the talk)
    - machinebox.io
    - https://keras.io/
    - HNews search https://hn.algolia.com/?query=tensorflow&sort=byPopularity&prefix&page=0&dateRange=all&type=story
    - https://github.com/vahidk/EffectiveTensorflow?
    - https://github.com/astorfi/TensorFlow-World
    - https://blog.chewxy.com/2016/09/19/gorgonia/
    - http://www.deeplearningbook.org/

## How to build SDKs (Dropbox experience)

Diwaker Gupta - [@diwakergupta](https://twitter.com/diwakergupta)

- Dropbox backend infra has been rewritten from Python to Go
    - talk in GopherCon https://www.youtube.com/watch?v=5doOcaMXx08
    - https://github.com/dropbox?utf8=%E2%9C%93&q=&type=source&language=go
- how to build SDKs:
    - coge generation: JSONSpec, IDLs+generators, OpenAPI/Swagger
    - principle of less surprise: no external or vendored deps
    - make config simple: 
        - go flags are too limited, avoid them
        - env vars: they are global state and difficult to test (???)
        - use a plain struct
        - don't use persistence or I/O, let that to the consumer
    - add verbosity levels / logging
    - how to:
        - json unions of different types
            - marhsaling: use `omitempty` tag
            - unmarshaling: use `json.RawMessage` and implement `json.Unmarshal` and then choose the correct type
        - inherited types
            - marshaling: use a dummy interface that returns a bool to assert if its a certain type or the other
            - unmarshaling: similar, struct embedding of the union and then implement `json.Unmarshal`, inside it another unmarshal to each union'ed type to see if has contents: :rainbow: ![](https://i.imgur.com/8dI1F6Z.jpg =500x)
    - do not auth
    - autogenerate tests, like aws-sdk-go which uses json to define them
    - do idiomatic error handling, pkg/errors is nice.
- download https://github.com/dropbox/dbxcli :smiley:

## WebSockets

Konrad Reiche - [@konradreiche](https://twitter.com/konradreiche)

- WS. 2008. Towards real time. REST is 20 years.
- REST maps HTTP verbs to Resources? Not just that.
    - defined in [Architectural Styles and the Design of Network-based Software Architectures](https://www.ics.uci.edu/~fielding/pubs/dissertation/fielding_dissertation_2up.pdf)
- gorilla/websocket is better than the stdlib one
    - gorilla's one reads full messages even they are in different frames. The stdlib one not, but they cannot change it. 
    - I think he does not know about https://github.com/gobwas/ws cause its brand new
- example code using gorilla's, not many notes because I've already done that in Social Point 3 years ago: http://go-sp.gonzaloserrano.io/#/27

## syscalls

Liz Rice - [@lizrice](https://twitter.com/lizrice)

- syscalls are used for: files / devices / processes / network / time-date
    - you can see them with strace
- package `syscall`
    - OS-specific files
    - autogenerated files 
- writing a strace-like tool in go