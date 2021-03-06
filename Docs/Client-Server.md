# Client-Server Programming

## Server Pattern

With a type and a module:

```fsharp
type Server =
  {opCh1: Ch<Request1>
   // ...
   opChN: Ch<RequestN>}

let create params =
  let chs = {opCh1 = Ch ()
             // ...
             opChN = Ch ()}
  // ...
  let rec loop state =
    // ...
    // "server" ops
    let op1 (*...*) = (*...*) chs.opCh1 (*...*)
    // ...
    let opN (*...*) = (*...*) chs.opChN (*...*)
    // ...
    match state with
     | (*...*) ->
       op? (*...*) <|> (*...*) <|> op? (*...*)
     | (*...*) ->
       op? (*...*) <|> (*...*) <|> op? (*...*)
  start <| loop state
  chs

// "client" ops
let op1 server (*...*) = (*...*) server.opCh1 (*...*)
...
let opN server (*...*) = (*...*) server.opChN (*...*)
```

With a class:

```fsharp
type Server (params) =
  let opCh1 = Ch ()
  // ...
  let opChN = Ch ()

  do // ...
     let rec loop state =
       // ...
       // "server" ops
       let op1 (*...*) = (*...*) opCh1 (*...*)
       // ...
       let opN (*...*) = (*...*) opCh2 (*...*)
       // ...
       match state with
        | (*...*) ->
          op? (*...*) <|> (*...*) <|> op? (*...*)
        | (*...*) ->
          op? (*...*) <|> (*...*) <|> op? (*...*)
     start <| loop state

  // "client" ops
  member server.op1 (*...*) = (*...*) opCh1 (*...*)
  // ...
  member server.opN (*...*) = (*...*) opChN (*...*)
```

## Call Patterns

### CommitOnReply: `unit -> Alt<Reply>`

```fsharp
server: opCh *<- Reply
```

```fsharp
client: opCh
```

This is a specialized pattern that should only be used when
* the reply is cheap to compute, because it is computed every time the operation
  is made available by the server, and
* there are no request parameters.

#### Example: Unique id server

```fsharp
type Id () =
  let reqCh = Ch ()
  do server << Job.iterate 0 <| fun i ->
       reqCh *<- i ^->. i+1
  member s.New = reqCh :> Alt<_>
```

### CommitOnRequest: `Request -> Alt<unit>`

```fsharp
server: opCh ^=> fun Request -> // ...
```

```fsharp
client: opCh *<- Request
```

This is a specialized pattern that should only be used when
* the operation cannot fail, because failure cannot be signaled to client.

#### Example: Stack

```fsharp
type Stack<'x> () =
  let pushCh = Ch ()
  let popCh = Ch ()
  do server << Job.iterate [] <| function
       | []   -> pushCh      ^-> fun x -> [x]
       | h::t -> popCh *<- h ^->. t
             <|> pushCh      ^-> fun n -> n::h::t
  member s.Pop = popCh :> Alt<_>
  member s.Push (x: 'x) = pushCh *<- x
```

Note that the above uses two distinct call patterns.

### CommitOnRequest: `Request -> Alt<Reply>`

```fsharp
server: opCh ^=> function Request replyIv ->
          // ...
          replyIv *<= Reply
```

```fsharp
client: opCh *<-=>- fun replyIv -> Request replyIv
```

### CommitOnReply: `Request -> Alt<Reply>`

```fsharp
server: opCh ^=> function Request (replyCh, nack) ->
          // ...
          replyCh *<- Reply <|> nack
```

```fsharp
client: opCh *<+->- fun replyCh nack -> Request (replyCh, nack)
```

### AsyncNoReply: `Request -> Job<unit>`

```fsharp
server: opCh ^=> function Request -> // ...
```

```fsharp
client: opCh *<+ Request
```

### AsyncWithReply: `Request -> Alt<Reply>`

```fsharp
server: opCh ^=> function Request replyIv ->
          // ...
          replyIv *<= Reply
```

```fsharp
client: opCh *<+=>- fun replyIv -> Request replyIv
```
