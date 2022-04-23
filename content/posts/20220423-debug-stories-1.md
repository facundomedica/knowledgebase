---
title: "Debug stories 1: darwin-arm64 Cosmos SDK builds are not compatible with other archs"
author: "Facundo Medica"
type: "post"
date: 2022-04-23T10:10:02-03:00
draft: false
subtitle: "Mac M1 builds are 99.9% compatible; not enough for consensus lol"
image: ""
tags: ["go","cosmos"]
---

> **TL;DR** Pointer addresses look different on different architechtures, so when converted to hex, they have different
lengths. This difference in length causes a difference in gas usage, which is used to produce a hash.

At Umee we were experiencing a memory leak in our testnet validator nodes, so to me the best way forward was to run a node locally on my Macbook Pro M1, run pprof, add a bunch of log statements and go from there. _Easy right?_

So I did all the node sync dance: downloaded the latest binary, the genesis.json and got the node list for my persistent peers setting. Started the node with `umeed start` and off we go! But...

```bash
Failed to process committed block (1005:4FB45CC400ADE7B6F640EBE90C51C5EB58C0161D65C9F0B56771914C2F2E6A7D):
wrong Block.Header.LastResultsHash.  Expected 520376AE1EC01E88011892D398C88ADA48EB941F6DC975DCA45125C4398A8D44,
got E30709AEEC492763438DA9DB0F64E9C69EFDAF237AE10778FFA4C03009347904
```

Ok... so... my local node doesn't agree with the chain? Maybe it's just a momentary issue, you know, _"turn it off and turn it on again"_. I `unsafe-reset-all` and restart the binary; but again, the same issue arises. At this point I just went ahead to a Linux ARM server that I have running with pretty much the same tooling I use locally and do all the same steps once more. This time it works!

But why would it work in a machine and not in another? The binary's source code was the same, the build process was the same, the only difference was the machine it was being run on. When a hash doesn't match you know for sure that there's some part of the data being hashed that doesn't match. This could meant that the entire data being hashed is different or that a single byte is different.

So yeah, I "[nerd sniped](https://xkcd.com/356/)" myself, left whatever I was doing and got to work on this. Finding indeterminism in a system that's meant to be **absolutely deterministic**? That sounds too much fun to ignore.

## Gathering data

First step was identifying the block that this was happening at. Easy, just look at the error message: `Failed to process committed block (1005:4FB...`. So this happened while processing the 1005th block. I went to check the block contents using a known node RPC (`http://.../block?height=1005`) and it was clear that `Block.Header.LastResultsHash` was different (no news here, just what the error message said :P).

But because the error is about `LastResultsHash` I figured that the actual error was happening a block before, so I ignored block 1005 and focused on **block 1004**. Ok, now we have to check the contents of block 1004, which contained 4 transactions:

- 3 txs with a single `AggregateExchangeRatePrevote` message each from Umee's Price Feeders.
- 1 tx with 2 messages: `ibc.core.client.v1.MsgUpdateClient` and `ibc.core.channel.v1.MsgChannelOpenTry` from an IBC relayer (Hermes in this case).

Main suspect here would be the `AggregateExchangeRatePrevote` message, given that it's from a brand new module we are currently testing. No way that very well-known messages are causing the issue right? 👀

## Follow the links and log absolutely everything

Next step is to add log statements **everywhere**.

> Visibility and fast trials are key when you are debugging code bases you are not very familiar with.

I have [my own process](/posts/2022/04/23/my-debug-process/) to do guerilla debugging, check it out, it might work for you too.

In this case I was aiming to find where `LastResultsHash` was being set. After some Googleing I found that `LastResultsHash` was being set here by the `ABCIResponsesResultsHash` function:

[https://github.com/tendermint/tendermint/blob/v0.34.16/state/execution.go#L450-L465](https://github.com/tendermint/tendermint/blob/v0.34.16/state/execution.go#L450-L465)

```go
	return State{
		Version:                      nextVersion,
		ChainID:                      state.ChainID,
		InitialHeight:                state.InitialHeight,
		...
->  LastResultsHash:              ABCIResponsesResultsHash(abciResponses), <-
		AppHash:                      nil,
	}, nil
```


Following that function we see it's pretty simple:

```go
func ABCIResponsesResultsHash(ar *tmstate.ABCIResponses) []byte {
	return types.NewResults(ar.DeliverTxs).Hash()
}
```

So I naturally did something like:

```go
func ABCIResponsesResultsHash(ar *tmstate.ABCIResponses) []byte {
        for _, v := range ar.DeliverTxs {
                log.Println(v.String())
        }
	return types.NewResults(ar.DeliverTxs).Hash()
}
```

I then repeated this in the Linux server and compared the outpus around the time I think it was failing with a tool like [this one](https://text-compare.com/). And after some time looking at it I found a difference!

```
My macbook: gas_used:177437
Server: gas_used:177371
Diff: 66
```

It was clear that there was a difference in gas used by one of the messages in that block, by looking at the transactions around there, I knew that it was caused by either `ibc.core.client.v1.MsgUpdateClient` or `ibc.core.channel.v1.MsgChannelOpenTry`. So I was like "ok, then I'll just check their types and find something evident like a an `int` instead of `int64`" (which is one of the main examples in which a different architecture can cause indeterminism).

But after a while of looking at the code I couldn't figure it out... At this point I was a bit stuck, so I decided to forget about the specific messages and dig deeper into how the gas is counted.

## Digging deeper

Next step was to check where the gas is being counted. I thought that if I could break down gas usage into each separate operation then I would be able to see differences. Never really went this far under Cosmos' hood, but I found after a while this file: [cosmos-sdk/store/gaskv/store.go](https://github.com/cosmos/cosmos-sdk/blob/v0.45.2/store/gaskv/store.go). Bingo! So in this file you can see a bunch of lines like this:

```go
gs.gasMeter.ConsumeGas(gs.gasConfig.ReadCostPerByte*types.Gas(len(key)), types.GasReadPerByteDesc)
gs.gasMeter.ConsumeGas(gs.gasConfig.ReadCostPerByte*types.Gas(len(value)), types.GasReadPerByteDesc)
```

What I did next is to add matching log statements to each of these calls (I actually first only printed the gas used and then added the key and value, but let's keep this post short lol):

```go
log.Println(gs.gasConfig.ReadCostPerByte*types.Gas(len(key)), types.GasReadPerByteDesc, string(key))
log.Println(gs.gasConfig.ReadCostPerByte*types.Gas(len(value)), types.GasReadPerByteDesc, string(value))
```

Then I ran the same code on the server and compared the outputs.

```
Mac
gas consumed, operation, key/value
1000 ReadFlat ibc/fwd/0x14000ffda48
63 ReadPerByte ibc/fwd/0x14000ffda48 <- +3
42 ReadPerByte ports/transfer
...
2000 WriteFlat ibc/fwd/0x140084adec8
630 WritePerByte ibc/fwd/0x140084adec8 <- +30
1380 WritePerByte capabilities/ports/transfer/channels/channel-0
...
1000 ReadFlat transfer/fwd/0x140084adec8
78 ReadPerByte transfer/fwd/0x140084adec8 <- +3
0 ReadPerByte 
...
2000 WriteFlat transfer/fwd/0x140084adec8
780 WritePerByte transfer/fwd/0x140084adec8 <- +30
1380 WritePerByte capabilities/ports/transfer/channels/channel-0
```

```
Linux server
1000 ReadFlat ibc/fwd/0x40010081a8
60 ReadPerByte ibc/fwd/0x40010081a8
42 ReadPerByte ports/transfer
...
2000 WriteFlat ibc/fwd/0x4005b534a8
600 WritePerByte ibc/fwd/0x4005b534a8
1380 WritePerByte capabilities/ports/transfer/channels/channel-0
...
1000 ReadFlat transfer/fwd/0x4005b534a8
75 ReadPerByte transfer/fwd/0x4005b534a8
0 ReadPerByte 
...
2000 WriteFlat transfer/fwd/0x4005b534a8
750 WritePerByte transfer/fwd/0x4005b534a8
1380 WritePerByte capabilities/ports/transfer/channels/channel-0
```

And here we see that there's an extra character on the software running on my Mac! And it matches the difference `gs.gasConfig.ReadCostPerByte = 3` and `gs.gasConfig.WriteCostPerByte = 30`, so 2 reads and 2 writes with an extra byte each sums up to 66 gas points.

## The root cause

After reading the logs above it was trivial to find the root of the whole issue, a quick search for /fwd/ got me to [this function](https://github.com/cosmos/cosmos-sdk/blob/v0.45.2/x/capability/types/keys.go#L39-L43) in the `x/capability` module:

```go
// FwdCapabilityKey returns a forward lookup key for a given module and capability
// reference.
func FwdCapabilityKey(module string, cap *Capability) []byte {
	return []byte(fmt.Sprintf("%s/fwd/%p", module, cap))
}
```

Here we can see that it converts the pointer to a string by using %p which is the `address of 0th element in base 16 notation, with leading 0x`. Which depending on the architecture seems to be longer or shorter given that some memory spaces might start at a different address (and I assume even in the same architecture depending on the amount of memory available, but I'm no expert so I might be extremely wrong).

So the issue was that it was assumed that the address' hex representation length was going to be exactly the same on all machines, but there's no way to guarantee this.

**Why no one found this before?** Simply because most of nodes are running Linux on an amd64 processor and people running nodes on Macbooks are the exception, probably just devs. And because when you are developing you are probably running all your nodes locally, there's no difference among them, so they all agree on the same address length.

## Conclusion

This was my first attempt at writing down my thought process while debugging stuff, and btw I've ommited many details and trials, I wished it was this straightforward 😆. I don't know the details to many things what I just commented, so yeah, making a lot of assumptions here :)