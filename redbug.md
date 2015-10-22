back to [main page](intro.md)

## intro ##

The erlang tracing (the so-called "trace bifs",
`erlang:trace/3` and `erlang:trace_pattern/3`) are insanely
powerful. However, they might be a bit hard to get used to.

For example, imagine this problem;

We have a system using 10,000 processes. Due to a bug, someone
occasionally sends the atom `true` to an important server process. This
eventually causes total system failure. We can not easily load code
onto the system since it sits at a customer site.

Here's a piece of code (that can be pasted into the Erlang
shell). It will trigger when someone sends `true`, and log the name
and current function of the sending and the receiving processes.

```
% pulls out process info for a given pid. 
Pi = fun (P) -> 
 try 
  {element(2,case process_info(P,registered_name) of 
              [] -> process_info(P,initial_call); 
              R -> R 
             end), 
   element(2,process_info(P,current_function))} 
 catch _:_ -> dead 
 end 
end.

% called by Init. receives trace messages, filters, logs the hits.
Loop = fun (G) -> 
  receive {trace,Sender,send,true,Rec} -> io:fwrite("~p -> ~p~n", [Pi(Sender), Pi(Rec)]); 
  _ -> ok end, 
 G(G) 
end.
% called by spawn. registers, starts the trace, calls Loop
Init = fun () -> 
  register(trip,self()), 
  erlang:trace(all,true,[send]), 
  Loop(Loop) 
end.
```

We start the tracer;
```
1>spawn(Init).
```

For clarity, I'll register a name for the shell;
```
2>register(luke,self()).
true
```

So let's send ourself a 'true';
```
3> self()!true.
```

The tracer will print this;
```
{luke,{shell,eval_loop,3}} -> {luke,{shell,eval_loop,3}}
```

`luke` (i.e. the shell) sent `true` to himself.

To be neat, I stop the tracer;
```
4>exit(whereis(trip),kill).
```

The point of the above was to show
  * the trace bifs have immense power
  * a higher-level interface is in order

## dbg ##
The standard OTP solution is called `dbg`. Alas, I must recommend
against using this on live systems, since it is way too easy to get it
wrong.

Try this for a laugh (it **will** kill the node).

`5> dbg:tracer(),dbg:p(all,m).`

Another problem with `dbg` is that it does not lift the level of
abstraction enough (from that of the trace bifs).

## things look grim, what to do? ##

redbug aims to be a safe and simple interface to the erlang trace
functionality, with some extra features thrown in.

The simplicity comes from exposing only the most useful (i.e. the bits
I use often) parts of the API in a concise way.

The safety comes partly from the simplicity, but also from the way the redbug process works.

redbug supervises itself and will kill itself if it deems that it is
in danger of overloading the node. It also delays the processing of
the trace messages until after the trace is done.

The most interesting extra feature is probably the `stack`
match spec. Answers the eternal question: "Who the #$%^ is calling my
function with #$%^ argument?"

## interface ##

The interface consists of the function `start`.

redbug:start(Time,Msgs,Trc[,Proc[,Targ]])

> Time: integer() [ms](ms.md)
> Msgs: integer() [#](#.md)
> Trc: list('send'|'receive'|{M,F}|{M,F,RestrictedMatchSpecs})
> Proc: 'all'|pid()|atom(Regname)|{'pid',I2,I3}
> Targ: node()
> > RestrictedMatchSpecs: list(RMS)

> RMS: 'stack'|'return'|tuple(ArgDescriptor)
> ArgDescriptor: '_'|literal()_

Trace for 5000 milliseconds or until 1 trigger on `erlang:phash`

`redbug:start(5000,1,{erlang,phash}).`

As above, but on `erlang:phash/2`

`redbug:start(5000,1,{erlang,phash,[{'_','_'}]}).`

As above, but second argument must be 16

`redbug:start(5000,1,{erlang,phash,[{'_',16}]}).`

As above, but show the return value

`redbug:start(5000,2,{erlang,phash,[{'_',16},return]}).`

As above, but show the call stack

`redbug:start(5000,2,{erlang,phash,[{'_',16},stack,return]}).`

The output of the last command looks like this

```
15:59:39 <prfTrc> {erlang,phash,[time,16]}
  {prfTrc,loop,1} 
  {prfTrc,start_trace,3}
  {dict,fetch,2}  
15:59:39 <prfTrc> {erlang,phash,2} -> 6
```

The trigger was
`erlang:phash(time,16)`
called in dict:fetch/2
and it returned 6.

## misc ##

The call stack displayed by redbug is really a call return stack;
i.e. only functions that will be returned to are shown. This is a
feature of the emulator (tail call optimization), and cannot be
circumvented in a safe way (as far as I know).

redbug has the idiotic/awesome feature that it can be run from a Unix shell.
actually pretty handy if you're working against an embedded system.

```
bash$ redbug shad@mwux005 3000 2 "{erlang,phash}"
16:26:00 <prfTrc> {erlang,phash,[time,16]}
```
