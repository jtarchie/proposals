# Logging Proposal

This a document for building a new logging platform. It hopefully captures the
problem, the intention, and minimal viable product/idea.

## Problem

All cloud applications need logging. It is useful to both capture and forward
logs to different sources. For example, forwarding messages to file for log
rotation or datadog for metric analysis.

When it comes to routing and storing logs in the cloud, there are many
implementations and proposals. From my experience, they always run into two
issues:

1. A minimal plugin architecture.

   In `fluentd`, the choice is between writing extensions in C or Ruby, which
   have to bundle with the release. In `rsyslog`, it is a proprietary config
   (turned programming) language. In `syslog-ng`, it is a proprietary config
   (turned programming) language.

2. Load handling, which is not to be confused with performance here. The ability
   to handle load is to ensure no message is lost. Performance can be a key part
   in this, but so is the ability to batch, queue, and ensure controls for
   stream pressure.

   Some logging systems choose to drop messages, so they never have to "catch
   up".

These solutions all require software to be installed on a server. They also
require knowledge of the user of how to configure them. If you have ever Googled
`filter messages on server`, you've been met with a plethora of StackOverflow
articles.

## Intention

Build a logging platform that contains the following:

- An embedded programming-language first approach. Avoid esoteric/custom/psuedo
  DSL config languages.
- Allow control of batching, queuing, and back/front pressure controls. There
  should be good defaults, never drop a message, but never block from receiving
  either. Users always need to adjust the dobs and nials.

NOTE: [fluentbit](https://fluentbit.io) has these some (or all?) these features,
but in a different architecture and intention.

## MVP

Using the modern [syslog protocol](https://tools.ietf.org/html/rfc5424), build
an application that can accept UDP/TCP connections and allow the user to forward
these messages. The forwarding capability will be limited to TCP/UDP and writing
to files.

The forwarding configuration would be defined in via configuration in an
embeddedable language. For the purpose of this document, let's consider Lua.

From a technology perspective, this could be done a few ways.

1. Golang with a Lua interpreter [1](https://github.com/yuin/gopher-lua)
   [2](https://github.com/Shopify/go-lua) With golang, it has the network and
   concurrency primitvies to handling this load. The Lua implementations are
   incomplete, but are usable.

   Opinion: Building servers is easy, building robust scalable servers is hard
   here. It will be fast, but so much server architecture would have to be
   built.

1. Elixir with a Lua interpreter [1](https://github.com/rvirding/luerl) Elixir
   is build on the Erlang VM. It also has networking and concurrency primitvies,
   with frameworks like [GenStage](https://github.com/elixir-lang/gen_stage) for
   building producer-consumer patterns. The Lua implementations are incomplent,
   but are usable.

   Opinion: Erlang has a proven track record for building scalable servers. It
   has well used frameworks, immutable data-structures, etc. It will not be as
   _fast_ as Golang, but extensions can be written in C/Rust. Erlang is also
   getting a JIT soon.

1. Lua with [Luvit](http://luvit.io/) Luvit is a IO framework for Lua (like node
   is to Javascript). Building it directly in Lua gives the advantage on
   one-language for everything.

   Opinion: Having _one language_ would be really powerful. Since luvit runs on
   LuaJIT, it will be fast out-of-the-box. It similar to golang, good network
   primitives, but requires building out serve architecture.

All the above support mechanisms for packaging and distributing a single
artifact.

### Forwarding a message

If a user wanted to configure a TCP server to receive messages on port 5555 and
forward them to `example.com:4444` using TCP.

```lua
-- create a client that message will be forwarded to
local forward_to = tcp_client:new("example.com:4444")

-- start a tcp server that will wait for syslog messages
tcp_server:new(":5555", function(message)
  forward_to:send(message)
end)
```

Starting the server, listening, accepting, and parsing messages is abstracted
away from the user. They will just receive a `message` struct (representing the
syslog message) for their use.

### Modifying a message

If a user wanted to modify a message in transit, for example to remove sensitive
data.

```lua
-- create a client that message will be forwarded to
local forward_to = tcp_client:new("example.com:4444")

-- start a tcp server that will wait for syslog messages
tcp_server:new(":5555", function(message)
  -- message is an object that can be modified in place
  message.hostname = "redacted"
  
  forward_to:send(message)
end)
```

The `message` struct can be modified in place. It will then be re-encoded for
the forwarding service.

### Batching

If a user wants to forward messages in batches of 100, instead of one-a-time.

```lua
-- create a client that message will be forwarded to
local forward_to = tcp_client:new("example.com:4444", {
  "batch" = 100,
})

-- start a tcp server that will wait for syslog messages
tcp_server:new(":5555", function(message)
  -- message is an object that can be modified in place
  message.hostname = "redacted"
  
  forward_to:send(message)
end)
```

This means that until the `send` is filled to a buffer of 100 messages for
forwarding service will not received anything.

There are many controls that could be exposed here instead of batch -- purge on
a time interval, byte size of buffer, etc.
