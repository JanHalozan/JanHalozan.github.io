---
layout:     post
title:      "Jarvis miniseries part VI: Intent execution"
subtitle:   "Getting our AI-powered home assistant in Rust to actually do things"
date:       2024-07-01 20:00:00
author:     "Jan"
header-img: "img/post-jarvis/bg_intents.jpg"
---

# Where we left off

Previously we coded Jarvis' brain which was far from trivial. But now we have a complete system that turns raw spech into something we can actually act upon.

## Goal of this post

<a href="/img/post-jarvis/jarvis_processing_pipeline_intent.png">
![Image of the processing pipeline](/img/post-jarvis/jarvis_processing_pipeline_intent.png)
</a>

What we'll achieve by the end of this post is to execute the `Intent::Command` piece of the intent enum. There is a lot of possibilities here since this is highly dependant on where you want Jarvis to run. For the purposes of this demo I'll run it on a RaspberryPI and interact with the environment via GPIO.

_This is going to be more of an explainer post than a fully fledged implementation of GPIO. Which also means it'll be a lot ligher!_

## Intent execution

When it comes to commands there really isn't that much to be done on the side of Jarvis. It needs to be able to interact with external environment somehow and we have a lot of freedom in terms of approaching this. As I've said I'll be using a RaspberryPI for my demonstration. I'll create a separate microservice for the actual GPIO manipulation and send requests to it via a simple message queue.

## Jarvis part

So instead of integration into `main.rs` being at the end I'll include it at the begging this time around.

```rust
let (executor_tx, executor_rx) = channel::<ClassifierOutput>();
thread_pool.spawn(async move {
    processing::intent_executor::main(classifier_rx, executor_tx);
    println!("Command executor shutting down");
});
```

And let's take a look at the executor `main`.

```rust
use std::sync::mpsc::{Receiver, Sender};

use crate::model::{command::Command, intent::Intent};

use super::classifier::ClassifierOutput;

pub fn main(classifier_rx: Receiver<ClassifierOutput>, executor_tx: Sender<ClassifierOutput>) {
    while let Ok(result) = classifier_rx.recv() {

        if let Ok(ref intent) = result {    
            match intent {
                Intent::Command(command) => execute_command(command),
                Intent::Question(_) => {}
            };
        }

        if executor_tx.send(result).is_err() {
            break;
        }
    }
}

fn execute_command(command: &Command)  {
    println!("Executing {:?}", command);
}
```

Could it really be that simple? Yes, it actually can. All Jarvis needs to do is to convert the `Command` part of the intent into a message and send it. The service on the other end is responsible for taking care of the actual execution.

**An important caveat** is present here. Which is if we state an instruction of which the result already matches the current state (ie. there's nothing to actually do) we won't know about it here. I didn't handle this intentionally because the point was to develop the processing pipeline. But if you wanted to support it there would need to be a mechanism to check if the current state matches the target on and based on that the `executor_tx` would send a different payload indicating a NOOP along which could then be interpreted in the feedback generator.

## Zeus

The counterpart to Jarvis will be Zeus. The god that actually makes things happen. I could go into a separate post on how to fully flesh it out but this series isn't about that. So I'll just add the main points. I've decided to write Zeus in Go for fun.

## Wrapping up

Now that we can actually turn on the lights and interact with other parts of the environment it's time to start preparing a response and providing vocal feedback to the user. We'll be looking at that next.