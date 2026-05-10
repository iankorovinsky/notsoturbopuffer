# Decisions

A rough collection of the decisions that I make surrounding the repo. This is meant to document decisions so that in the future, I can remember/understand why I did things. It also contains some strong opinions, loosely held, on:
    a) Some of the UX of the tpuf API. I'm sure a lot of these decisions were made in favour of latency/performance, but since this database is not so performant, I'm prioritizing a UX I enjoy. A lot of the decisions are made off of the tpuf systems, and it is used as a constant source of inspiration.
    b) The overall product experience of using a DB like this. I want this to be a nice db (which can also be run locally) which tpuf does not currently offer (for good reasons). Since I am a broke college student with no cloud credits, I need something that can run locally in the near future.

## Local vs. Cloud

I'm going to start with making a version of this that runs locally (within this repo). It will not offer many of the hardware / caching advantages that tpuf offers. However, it is much cheaper to create, and I will follow many similar principles of object-based storage.

## Lack of Vectorization + Rerank Support

Drawing inspiration of libraries like Tinker, I like the fact that you're able to fine-tune models without ever having to use use massive compute on your machine. I want to figure out if there's any way to do this well + fast locally.