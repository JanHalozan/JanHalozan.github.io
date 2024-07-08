---
layout:     post
title:      "Jarvis - An AI powered home automation assistant in Rust"
subtitle:   "An Iron Man inspired home automation assistant from scratch."
date:       2024-07-07 20:00:00
author:     "Jan"
header-img: "img/post-jarvis/bg.jpg"
---

# Introduction

Ever since I saw the first Iron Man movie and got captivated by his human-like smart assistant Jarvis, I've wanted to replicate it. I saw the movie sometime around 2009, so a long time has passed since then. I never fully got around to actually building it... until now. Fortunately, a lot of great advances have been made since that time, and the complexity of building something like that has gone down significantly. There's already plug-and-play solutions out there, but I didn't let that discourage me since I just wanted to solve the engineering challenge of building one. So let's get down to it!

What we're building is obviously not going to be anywhere near Iron Man's Jarvis, but it will be able to execute commands we define for it and possibly answer a question or two. It's going to be very extendable so that modifications and upgrades can be done easily. Here's what the final product looks like:

`<Insert video here>`
I'll add a video of it working soon™. I promise.

_This is going to be a multipart series since there are quite a few pieces to cover, but rest assured I've actually built the thing first. Which means I won't abandon it in the middle._

## Requirements & Design choices

I'm going to name my assistant **Jarvis** as well to pay homage to where the idea originated. What I want Jarvis to do is pretty simple:
- Take commands via speech
- Recognize commands and determine if they're supported in our home
- Execute the given command
- Provide audio feedback after acting on a command
- Possibly some extra nuggets here and there that I'll show off as we go

Another important thing to decide is what to build the project with. I've always liked performant languages, and in my early days, I learned coding with C and C++, so I have a good understanding of how things work under the hood. I've been fiddling with Rust a lot lately, and I absolutely love working with it, so I've decided to use **Rust** for building Jarvis.

_A lot of the approaches I'll describe in this post are language-agnostic, so you could easily code a similar project in a different language if you decide._

## High level architecture

We'll break Jarvis down into a lot of small components which are sequentially tied together and form a complete processing pipeline. That way, it's extendable and can be easily modified or adapted for other needs. Here's the high-level diagram on how the pipeline will fit together:

<br/>
<a href="/img/post-jarvis/jarvis_processing_pipeline.png">
![Jarvis processing pipeline](/img/post-jarvis/jarvis_processing_pipeline.png)
</a>
<br/>

As these are nice separate blocks where each has its own concerns and functionality, I'll similarly write blog posts covering one block at a time. But first, let's look at what each block actually does (this is a very high-level overview; more detail on each block will be in its corresponding blog post):

- **Microphone listener**  
    As the name implies, this block will continuously listen to the microphone. It's where the pipeline starts since Jarvis is a voice-activated system.
- **VAD Chunker**  
    The microphone will give us a continuous stream of raw audio data, which isn't very useful for any recognition. The VAD (Voice Activity Detection) chunker will take the raw audio, process it to make sure there's actually useful audio in there, and then chunk it into separate pieces.
- **Wake word detector**  
    Because we don't want Jarvis to chime in on every conversation that might be happening around it, we'll add a wake word detector. Its job is to recognize if the audio frame contains the wake word and only then pass on the audio data onwards in the pipeline.
- **Recognizer**  
    The recognizer's job is pretty simple on a high level: take an audio frame and process it into a string of text. We'll use an AI model to accomplish this.
- **Classifier**  
    Once we have speech phrases, we'll need to figure out if there's anything useful inside them. To do this, we need to define the commands that Jarvis supports and find a way to classify which command (if any) is being triggered and create the Intent. Another job for AI, it seems.
- **Intent executor**  
    Once a command is classified and recognized, we'll call it an `Intent`. Intents are fully structured pieces of data that Jarvis supports and can act upon. The executor stage is meant to actually trigger what is supposed to happen (e.g., turn on a light).
- **Feedback generator**  
    We'd like Jarvis to talk back to us so that we know what's happening. It will either confirm what happened or explain what went wrong. The feedback generator is responsible for composing the text that will be conveyed back to the user.
- **Speech synthesizer**  
    The very last block which finally converts the feedback text into an audio stream that is played through the speakers.

And it's as simple as that :)

## Let's code!

Instead of just having an empty explainer post, I wanted to do some coding in it as well. What we'll do is set up the project and prepare the building blocks so that once we start working on the aforementioned processing pipeline, we can focus solely on that.

Since the project will be in Rust, we can use `cargo` to create a blank project. We'll do that by using `cargo new jarvis`. It creates the basic folder structure and gives us the manifest.

### Project Structure

On a high level, the folder structure in `src` I'll use is:
- `processing` - files for our processing pipeline. Each will cover one step of it.
- `core` - all of the shared functionality that can be reused.
- `errors` - custom errors.
- `traits` - trait definitions.

_Note: There are bits of Rust that aren't immediately intuitive if you've just started using the language. This series isn't meant to teach Rust, so if anything is unclear, the Rust documentation is an excellent resource. But you're welcome to reach out if anything is unclear._

### Set up

The more I investigated how various pieces would fit together to form the pipeline shown in the above picture, the clearer it became that it’s easiest to create this behavior by having a separate thread for each stage. This way, each stage can run asynchronously and communicate with each other sequentially (the output from one stage goes into the input for the next).

I'll use the hugely popular [Tokio library](https://tokio.rs/) for my async needs. Tokio allows us to do a few useful things:
- Have our `fn main` be async.
- Spawn synchronous or asynchronous threads and wait for their completion.
- Streamline access to the `Ctrl+C` termination signal.

### On threading and channels

To have each of the processing stages in its own thread, we can use `tokio`'s `spawn` or `spawn_blocking`. More on spawning in a second. However, to communicate with each other, threads also need a mechanism. With Rust's approach to memory safety, this initially feels like a non-trivial matter. However, fret not, as [`channels`](https://doc.rust-lang.org/rust-by-example/std_misc/channels.html) are here to the rescue. I opted to use the system `std::sync::mpsc` channels instead of Tokio ones as they cover all my needs and make the potential removal of Tokio as a dependency easier if needed.

So let's code our main function so that some of the patterns emerge:

```rust
#[tokio::main]
async fn main() {
    // Create a channel that will send data from one thread to the other
    let (mic_tx, mic_rx) = channel::<String>(); // The data type will change later

    // Spawn an async thread and move the `mic_tx` part of the channel into it
    tokio::spawn(async move {
        // Capture mic input somehow
        let data = "Hello from microphone".to_string();
        mic_tx.send(data).unwrap();
    });

    // Another thread for consuming microphone input
    tokio::spawn(async move {
        loop {
            let value = match mic_rx.recv() {
                Ok(item) => item,
                Err(_) => break,
            };
            println!("Received mic data '{}'", value);
        }
    });

    println!("Exiting");
}
```

This code looks like it makes sense, but upon closer inspection, there are a few flaws. The biggest one is that nothing really prevents our program from exiting since the threads run asynchronously.

The output might look like:

```
Exiting
Received mic data 'Hello from microphone'
```

Or it might not print the mic data line at all. That's because when the main thread of the program terminates, it'll kill all of the children threads as well.

We can use Tokio's [ctrl+c](https://docs.rs/tokio/latest/tokio/signal/fn.ctrl_c.html) signal to wait until we want to terminate the program. It'll almost start looking like an actual program after that. The second thing I want is to terminate it if any of the threads spawned encounter an unrecoverable error. For instance, if we cannot get a microphone to capture data from, there's no use in running Jarvis.

### Cross thread error handling and termination

There can be an error that happens in any of our processing stages at any time. Or we can decide to terminate the program ourselves. I'd quite like to see each thread terminate gracefully, and the program to wait until all threads have shut down before exiting.  
For waiting on all threads, we can use Tokio's `JoinSet`, which provides the same `spawn` and `spawn_blocking` functions but captures thread handles and groups them neatly together. We can wait for their exit by using:

```rust
while let Some(_) = thread_pool.join_next().await {}
```

So now the last piece is handling errors from other threads. I've decided to use a common structure that will group all of the potential signals we might need. And to make handling different kinds of errors more manageable, I'm using the [anyhow library](https://docs.rs/anyhow/latest/anyhow/).

I've created a `core/jarvis_signals.rs` file shown below. We'll use the `core` folder for shared and core functionality.

```rust
use std::sync::atomic::AtomicBool;
use anyhow::Error;

// Define a struct that will hold all of our signals
// right now it just has a single signal which tells
// us if we should shutdown
pub struct JarvisSignals {
    // We use an atomic bool since we'll be checking this value from multiple threads
    shutdown: AtomicBool
}

impl JarvisSignals {
    pub fn new() -> Self {
        JarvisSignals {
            shutdown: AtomicBool::new(false)
        }
    }

    // We return the value of our `shutdown` bool.
    // If you're curious about ordering more can be found here
    // https://doc.rust-lang.org/std/sync/atomic/enum.Ordering.html
    // but in a nutshell `Relaxed` means we don't care about memory
    // synchronization as we're only interested in the value
    // and we'll be checking this repeatedly in a loop
    pub fn is_shutdown(&self) -> bool {
        self.shutdown.load(std::sync::atomic::Ordering::Relaxed)
    }

    // We set the shutdown signal and if there was an error that
    // caused the shutdown we print it out.
    pub fn set_shutdown(&self, reason: Option<Error>) {
        if let Some(error) = reason {
            eprintln!("Terminating due to an unexpected error: {:?}", error);
        }

        self.shutdown.store(true, std::sync::atomic::Ordering::Relaxed);
    }
}
```

With `JarvisSignals` in place and some knowledge about `JoinSet`, we can fully flesh out and wrap up our `main.rs`:

```rust
mod core;

use std::{sync::{mpsc::channel, Arc}, time::Duration};
use core::jarvis_signals::JarvisSignals;
use tokio::{signal, task::JoinSet};

#[tokio::main]
async fn main() {

    // Our main signals struct, we need Arc to be able to
    // share it between multiple threads.
    // More details: https://doc.rust-lang.org/rust-by-example/std/arc.html
    let signals = Arc::new(JarvisSignals::new());
    let mut thread_pool = JoinSet::new();

    let (mic_tx, mic_rx) = channel::<String>();
    let mic_signals = signals.clone();
    thread_pool.spawn(async move {
        // The mic input will be a continuous stream of data eventually
        while !mic_signals.is_shutdown() {
            // Some dummy text from the mic thread every 0.5s
            let data = "Hello from microphone".to_string();
            mic_tx.send(data).unwrap();
            std::thread::sleep(Duration::from_millis(500));
        }
        println!("Mic input thread shutting down");
    });

    thread_pool.spawn(async move {
        // We don't need to check signals here due to the fact
        // that when the mic_tx is dropped the `recv()` will give 
        // us an error which will break the loop
        loop {
            let value = match mic_rx.recv() {
                Ok(item) => item,
                Err(_) => break
            };

            println!("Received mic data {}", value);
        }
        println!("Mic processing thread shutting down");
    });

    // Finally we need another thread that continuously checks if 
    // tokio::signal::ctrl_c is pressed or otherwise just chills for a
    // 100ms. When ctrl+c is pressed it tells the `signals` struct
    // to flip the shutdown bool to true and set None as the
    // termination error since this was an explicit user action
    let shutdown_signal = signals.clone();
    thread_pool.spawn(async move {
        loop {
            tokio::select! {
                _ = signal::ctrl_c() => {
                    shutdown_signal.set_shutdown(None);
                    break;
                },
                _ = tokio::time::sleep(Duration::from_millis(100)) => {
                    if shutdown_signal.is_shutdown() {
                        break;
                    }
                }
            }
        }
    });

    println!("Jarvis up and running");

    // Block the main thread until shutdown signal is received
    while !signals.is_shutdown() {
        std::thread::sleep(Duration::from_millis(100));
    }

    println!("Terminating auxiliary threads...");

    // Wait for child threads to terminate gracefully
    while let Some(_) = thread_pool.join_next().await {}

    println!("\nAux threads terminated. Exiting...");
}
```

Running the program now should print something similar to this:

```
Jarvis up and running
Received mic data 'Hello from microphone'
Received mic data 'Hello from microphone'
Received mic data 'Hello from microphone'
^CTerminating auxiliary threads...
Mic input thread shutting down
Mic processing thread shutting down

Aux threads terminated. Exiting...
```

So until we press ctrl+c, it will keep printing the mic string and then terminate.

## Wrapping up

We've now got a good foundation set up that allows us to delegate processing to different threads and wind them down gracefully when the user decides to terminate the app or an unhandled error happens. [In the next post](https://janhalozan.com/2024/07/01/jarvis-part-1-microphone/), we'll start looking into how to actually capture microphone input and go about processing the raw audio data.
