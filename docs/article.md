---
title: One dropdown cost 390ms on the same phone call
published: false
tags: voiceai, webdev, telephony, performance
cover_image:
canonical_url:
---

I work as a forward deployed engineer at a carrier, which means I spend most of my week on calls with teams building voice agents. The same conversation keeps happening. They have tuned their model hard, the demo still feels slow, and nobody has looked at the phone call underneath it.

So I measured one. Sixteen calls from a Telnyx number to my own mobile, changing one setting between them and nothing else.

**One dropdown costs you 390ms on every call your agent makes.** Not a model change, not a prompt, not a vendor migration. One setting most people never open.

![Four cities, 16 calls, 390ms between the fast pair and the slow pair](https://raw.githubusercontent.com/shahin-hu/dialtrace/main/docs/anchorsite.png)

## What the setting is

Every voice platform has to decide which datacentre handles the audio on your call. Telnyx calls it **AnchorSite**. The default is `Latency`, which picks for you, and you can pin it to a specific city instead.

I pinned it to four cities in turn and measured the gap between the caller answering and hearing a sound.

| City | Median | Individual calls |
| --- | --- | --- |
| Ashburn, VA | **650ms** | 640, 640, 660, 760 |
| Amsterdam, NL | **670ms** | 540, 620, 720, 780 |
| San Jose, CA | **1,060ms** | 640, 980, 1,140, 1,260 |
| Sydney, AU | **1,040ms** | 920, 980, 1,080, 1,500 |

Two cities land around 650ms and two around 1,050ms. Same caller, same destination, same codec, same everything else.

## Is that real, or am I reading noise?

Fair question, and my first attempt at this was noise.

I started with **one call per city**, which showed an 840ms spread and a tidy monotonic gradient from Ashburn down to Sydney. It looked great. Four calls per city destroyed it: the gradient vanished, Sydney stopped being the worst, and the spread halved.

Comparing any single city against another is four calls against four calls and reads as noise. Grouped as near against far it separates cleanly:

- Near, meaning Ashburn and Amsterdam: median **650ms**
- Far, meaning San Jose and Sydney: median **1,040ms**
- **7 of the 8** far calls were slower than every single near call
- A permutation test on the difference of medians gives **p = 0.012**

So the effect is real and the number is 390ms, not 840ms. If I had published the first version I would have published a number that was mostly luck.

## The bigger number nobody measures

Here is the part I did not expect.

**None of those sixteen calls had any AI in them.** No speech to text, no language model, no text to speech. Nothing was running. The call connected and a `speak` command played one word.

Across all seventeen calls I made that day, the median wait before the caller heard anything was **760ms**, and every single call was over half a second.

![17 calls, median 760ms, all over half a second](https://raw.githubusercontent.com/shahin-hu/dialtrace/main/docs/silence.png)

Teams budget around 800ms for a complete conversational turn. If the phone call alone costs 760ms before a model is asked anything, that budget was never real.

## Where the time actually goes

On one call I traced it properly, and **350ms of the wait was simply the carrier telling my laptop that the call had been answered**.

Think about what your agent does at that moment. The call connects. The carrier notifies your server. Your server works out what to say. Only then does it tell the carrier to speak. You pay for that round trip on every single call, and it is on the critical path.

That is also why the anchorsite matters. Moving the media anchor moves where that round trip has to travel.

Every voice AI benchmark I can find measures speech to text, the model and text to speech. None of them measure the call those three run on.

## Three things in the OpenAPI spec that are not in the docs

I went looking in Telnyx's own API specification while building the tool and found three things worth knowing. I work there, and I still found these by reading the spec rather than the documentation.

**Every `call.hangup` webhook already carries call quality data.** The field is `call_quality_stats`, and it holds MOS, jitter variance and packet counts split by direction. My call came back at **MOS 4.50 with 1.14% packet loss**. If you already consume these webhooks, that has been sitting in every call you have ever made and you have probably never opened the field.

**The Call Control API sends no ringing event.** An outbound call goes `call.initiated`, then `call.answered`, with nothing between. So post-dial delay cannot be separated from the seconds somebody spends reaching for their phone. TeXML status callbacks do have ringing. Call Control does not.

**`call.initiated` ignores your per-call webhook override.** If you pass `webhook_url` on the dial, that override applies only to *subsequent* webhooks. `call.initiated` is not subsequent, so it goes to the application's own `webhook_event_url` and you silently never see it. One word in the API reference is the only clue, and it cost me an entire measurement run before I understood it.

## What I could not measure

Two caveats, because the numbers are worth nothing without them.

**There are two clocks.** Mine stamps my own markers and the carrier's stamps the webhooks. I estimate the offset from the dial round trip and print an error bar beside it. On the call above it came out at **197ms ±242ms**. The error bar is wider than the number, which means I do not actually know the offset, and the tool says so rather than rounding the doubt away.

**My webhooks arrived through a tunnel**, which adds hops. Your own numbers should come in lower than mine. The tunnel was identical across all sixteen anchorsite calls, so it cannot explain a difference between them, but it does inflate the absolute figures.

## What I got wrong

Two things, beyond the n=1 gradient above.

I counted ring time as latency. It was **15.6 seconds of a 26 second call**. That is somebody walking to their phone, not something anyone can engineer away, and averaging it into call latency is how teams spend a sprint optimising something that was never theirs to fix. The tool now reports it separately and excludes it from any share-of-budget maths.

I also shipped a post-hangup grace window of 400ms. The cost webhook arrived **382ms** behind hangup on the first real call, so catching it at all was luck rather than margin.

## Reproduce it

The tool is MIT licensed, has no dependencies, and needs no account to see the output:

```bash
npx dialtrace demo
```

To measure a real call you need a Telnyx API key and a Call Control App:

```bash
export TELNYX_API_KEY=KEY...
export TELNYX_CONNECTION_ID=...

npx dialtrace call \
  --to +31612345678 \
  --from +12125550100 \
  --say "testing one two three" \
  --sync-app-webhook \
  --svg call.svg
```

It puts the carrier's events and your own pipeline stages on one waterfall. Your stages get there by posting a label to a local endpoint while the call runs:

```bash
curl -s localhost:8787/mark \
  -H 'content-type: application/json' \
  -d '{"label":"llm.done"}'
```

![A dialtrace waterfall of a single real call](https://raw.githubusercontent.com/shahin-hu/dialtrace/main/docs/real-call.png)

Only `src/telnyx.ts` knows anything about Telnyx, and it is about 80 lines of `fetch`. Another provider that emits call progress events with timestamps fits behind the same interface.

Code and the raw data for every call above: **[github.com/shahin-hu/dialtrace](https://github.com/shahin-hu/dialtrace)**

## If you run it

I would like to know two things. What your answer-to-first-audio comes out as, and which anchor you are on. Mine was 760ms with nothing in the path at all, and I have no idea yet how wide the spread gets across other people's setups.

There is also one question I have not answered. The default `Latency` setting picks an anchor for you, and I have only two calls on it, so I cannot tell you whether it picks well. That is the next measurement.
