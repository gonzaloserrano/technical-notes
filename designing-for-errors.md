# short dive into designing for errors

## railway oriented programming

> unhappy paths are requirements too

- [@ScottWlaschin](https://twitter.com/ScottWlaschin)
- https://vimeo.com/113707214

It's all about composition. You want to do a computation which is composed in steps of serveral computations (functions) that you compose.

So it's related to FP since you need some higher abstractions (functions) that will use your functions, and at the same time they need to be designed in a certain way to return result or error. 

Why would you do that? To avoid writing the error handling code everywhere and make your code look like you just wrote the happy path.

So in the railway there are two tracks:
- the sucess or happy track
- the error track

The results of your funcs are pattern-matched, so when error you will go to the error track, otherwise you continue in the happy track.

Steps to handle error in a FP way:
- create a result type (he uses the Either monad in F#), e.g:
```
Type TwoTrack<'T> =
    | Success of 'T * Message list
    | Failure of Message list
```
- create a bind function to convert error throwing functions to two-track functions
- compose the two-track functions together
- type your errors
     

### monads

Algebraic + types stuff. I don't know much more :-)

Basic ones that are interesting for handling errors:
- maybe
- either: similar to Maybe but returns a value instead of Nothing

[maybe vs exceptions]( https://softwareengineering.stackexchange.com/questions/150837/maybe-monad-vs-exceptions)
  - use of optional
  - more semantic

## errors

> Documentation is everything that can go wrong
> 
> Test again error codes, not strings

- Model errors as types
  - having those then they can't go out of date in documentation because they are in the code
  
## what about go

I care about it because it's what I write in Go and finding ways of have a cleaner code is something of my interest.

- Go does not have inheritance, it's all about composition
- functions are 1st class citizens in Go 
  - https://golang.org/doc/codewalk/functions/
  - https://dave.cheney.net/2016/11/13/do-not-fear-first-class-functions
- returning `result, error` is a common signature for functions in Go
  - [errors vs exceptions](https://dave.cheney.net/2015/01/26/errors-and-exceptions-redux) by Dave Cheney

### implementation

Looks like Go is not very suitable for implementing monads in a general way.

https://groups.google.com/forum/?nomobile=true#!topic/Golang-Nuts/iIAhrtK8XRo 

> Again, it's incredibly awkward in Go. (It would actually be very awkward even in Haskell without some syntactic support). The basic idea is that each step in the sequence can only proceed if all previous steps have succeeded. 
  Note how the error returned does not describe *which* step failed; that's a problem with this approach in general - it assumes that an unadorned  error is sufficient diagnostic context. Remember that, the next time someone suggests that the Haskell Either monad is a better approach than Go's explicit error checking. 
  
Some attempts: 

- https://github.com/dc0d/rop/blob/master/rop.go
- https://play.golang.org/p/9fZ467PC_c
- https://github.com/SimonRichardson/wishful
- https://play.golang.org/p/P5DDZrcXZB
- https://godoc.org/labix.org/v2/pipe
- https://github.com/chewxy/lingo/blob/master/dep/nn2.go
- Go2 Result type proposal https://github.com/golang/go/issues/19991 (refused)

> It's domain-specific, yes, but in their general form, monads really don't give you any value that's not trivially implemented. 
The basic approach, though, where you can combine user-level  functions with higher-level control flow, is very useful. 

So it looks that the best thing you can do i Go is to re-implement a Monad-like approach with the concrete types you need:
- type your Result and errors
- abstract your logic in functions that return `(Result, error)`
- create a high-order function to pipe/chain/railway them
  - you can implement middlewares for other things like logging, tracing etc.
  - or the try something similar to the Maybe, Either, Option monads but applied to your types.
  
#### The official goblog example

https://blog.golang.org/errors-are-values by Rob Pike. 

And an interesting 3rd party follow-up blog post https://www.innoq.com/en/blog/golang-errors-monads/ concludes:
> Two commonly perceived problems of Go are that handling errors is verbose and repetitive and that parametric polymorphism is unavailable[6].
One of the authors of Go offers a solution to one of those problems, but his advice boils down to “use monads,” and because of the other problem you cannot express this concept in Go.
> This leaves us having to implement artisanal one-off monads for every interface we want to handle errors for, which I think is still as verbose and repetitive.

#### Another approach

Really interesting talk, must review again:
https://speakerdeck.com/rebeccaskinner/monadic-error-handling-in-go