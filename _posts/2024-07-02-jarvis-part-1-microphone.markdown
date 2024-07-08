---
layout:     post
title:      "Jarvis miniseries part I: Microphone input"
subtitle:   "Getting started with the processing pipeline and using cpal for capturing microphone input for our AI-powered home assistant in Rust"
date:       2024-07-01 20:00:00
author:     "Jan"
header-img: "img/post-jarvis/bg_mic.jpg"
---

# Where we left off

In the previous installment of this series we covered the goals and motivation behind this project, then set up some initial boilerplate code that allows us to run code on different threads and link them together.

## Goal of this post

<a href="/img/post-jarvis/jarvis_processing_pipeline_mic.png">
![Image of the processing pipeline](/img/post-jarvis/jarvis_processing_pipeline_mic.png)
</a>

In this part we're gonna start with the first building block of the pipeline: microphone input. It's pretty straightforward but does involve some setup and some code to get working.

## Microphone input

Capturing the mic is where the entire processing journey starts. I've briefly explained in the previous post that `processing` is where our code for processing will go and `core` will be used by shared functionality like `JarvisSignals`.

To capture the raw microphone input we'll use a library called [cpal](https://github.com/RustAudio/cpal). It's performant, written in pure rust and has a good community behind it. We'll use more packages from the same community for some other tasks in the future as well.

Without further ado let's prepare our microphone processing step. I've created a file called `microphone_listener.rs` in `processing`:

```rust
// Don't get confused with another `main` name - this one is in a different namespace.
pub fn main(signals: Arc<JarvisSignals>, microphone_tx: Sender<Vec<f32>>) -> Result<()> {
    Ok(())
}
```

I've decided to call each of the processing steps' main function `main` since they will perform the main piece of work for each step. Rust allows us to have multiple same function names due to namespacing.  
The function doesn't look too exciting right now. The only thing to note is that it takes two parameters:
- `signals: Arc<JarvisSignals>` signals are used to determine when to actually stop listening. When our mic will be enabled it will keep spewing data and won't stop until we explicitly tell it to. To do that we need some way of knowing when to stop - we've got our `is_shutdown()` ready for exactly this purpose!
- `microphone_tx: Sender<Vec<f32>>` a [`mpsc::channel`](https://doc.rust-lang.org/std/sync/mpsc/fn.channel.html) that will transmit (hence tx) the values we receive from the mic further to the next pipeline stage. All of the next pipeline steps will take a `Receiver` and a `Sender` to transform the data in their way and send it along the pipeline.

It returns an `anyhow::Result<()>` to inform us of any errors that might emerge when setting up the microphone or just return an empty tuple letting us know that everything is good.

### cpal

To start capturing data we'll need to construct a `cpal::Stream`. Let's see how to do that:

```rust
let stream = cpal::default_host() // We wanna get the system mic
    .default_input_device() // so we just take the default input device
    .ok_or(JarvisError::no_mic())? // We'll look at the JarvisError in a sec
    .build_input_stream(
        &INPUT_STREAM_CONFIG, // wat
        data_callback, // more wat
        error_callback, // even more wat
        None)?; // None we can deal with
```

You can look at the exact documentation for each step of the stream set up in cpal documentation. But in short we first take the default host (our computer) and ask for the default input device (usually the built in mic or the first one that is available). If there was an issue getting a mic we don't have anything available to capture data from. Jarvis can't run if it can listen to anything - meaning we've hit an unrecoverrable error.

### JarvisError

I've created a custom `JarvisError` struct in `errors/jarvis_error.rs` that looks like so:

```rust
use std::{error::Error, fmt::{Debug, Display}};

// The reason that caused this error
#[derive(Debug)]
pub enum JarvisErrorReason {
    NoMicrophone
}

// The main error struct, it could have more context and data
// associated with it in the future
pub struct JarvisError {
    reason: JarvisErrorReason
}

// Some shorthand functions so that we don't have to construct
// structs in our code directly
impl JarvisError {
    pub fn no_mic() -> Self {
        JarvisError {
            reason: JarvisErrorReason::NoMicrophone
        }
    }
}

// Implementation of Display and Debug traits so that we can print
// this error in the console 
impl Display for JarvisError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        let message = match self.reason {
            JarvisErrorReason::NoMicrophone => "No microphone found.",
        };

        write!(f, "{}", message)
    }
}

impl Debug for JarvisError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "Jarvis error \{\{ reason: {:?}, message: {} \}\}", self.reason, self.to_string())
    }
}

// And finally the conformance to std::error:Error
impl Error for JarvisError {
}
```

Phew! That's a big block of code just to tell Rust what went wrong. However it's a very flexible block of code. We could easily add more cases to the enum and propagate all kinds of errors. 

### Back to cpal

We had this block of code we were working on:

```rust
let stream = cpal::default_host() // This is now clear
    .default_input_device() // Clear also
    .ok_or(JarvisError::no_mic())? // Makes sense too
    .build_input_stream(
        &INPUT_STREAM_CONFIG, // still wat
        data_callback, // still more wat
        error_callback, // still even more wat
        None)?; // None we can deal with
```  

#### Stream Config
I don't wanna re-explain what is already written in the documentation on how to construct a stream so let me provide the actual implementations of those magic parameters.

```rust
const INPUT_STREAM_CONFIG: StreamConfig = StreamConfig {
    channels: 1, // We want our microphone stream to be mono
    sample_rate: SampleRate(48000), // 48k samples per second
    buffer_size: cpal::BufferSize::Default // Not too concerned with the buffer as we'll handle it ourselves
};
```

The config just specifies the sampling rate and some additional parameters.

#### Error callback

Error callback is pretty straightforward too:
```rust
let error_signals = signals.clone();
let error_callback = move |error: cpal::StreamError| {
    match error {
        cpal::StreamError::DeviceNotAvailable => error_signals.set_shutdown(Some(error)),
        _ => eprintln!("Capture stream error {:?}", error)
    };
};
```

We give `error_callback` a copy of `JarvisSignals` because if it catches a `DeviceNotAvailable` it means there's no way to get mic input anymore. Could be that Jarvis became self sentient and decided no longer to listen and just do or the user pulled out the mic cable out. Both are equally likely. In any other case we just print out the error and hope things will get better. We could be more sophisticated here if needed.

#### Data capture

Now for the actual data capture:

```rust
let data_signals = signals.clone();
let data_callback = move |data: &[f32], _: &InputCallbackInfo| {
    if data_signals.is_speaker_active() || data_signals.is_shutdown() {
        return;
    }

    let resampled = resample_audio(data, 48000, 16000);
    
    if let Err(e) = microphone_tx.send(resampled) {
        // If we can't propagate mic anymore it doesn't make sense to stay alive
        data_signals.set_shutdown(Some(e.into())); 
    }
};
```

What's this? Even more magic functions and variables that weren't explained before? Fret not - it's actually simpler than it might look. As I've explained in the first post of this series I've actually built Jarvis before writing any blogs posts so I've encountered and fixed all sorts of issues. The first of which was that it makes no sense processing microphone input if Jarvis is speaking. Another good use case for JarvisSignals. I've added an `speaker_active: AtomicBool` to it and implemented two functions `is_speaker_active(&self) -> bool` to check it and `pub fn set_speaker_active(&self, active: bool)` to toggle it.

Firstly we check if the speaker is active or if we're in shutdown. If we are we just return early from the callback ie. discard data. But this doesn't cause anything to shutdown just yet. We're just ignoring whatever comes into the closure.

Second we resample the audio. Why? Because a pipeline step further down wants the audio to be in 1 channel but at 16k sampling rate. More on that in a subsequent post. The way resampling works can be seen below:

```rust
fn resample_audio(input: &[f32], input_rate: usize, output_rate: usize) -> Vec<f32> {
    let factor = input_rate / output_rate; // by what factor are we downsampling. We could use a check here to ensure the factor actually makes sense 
    let cutoff = output_rate as f32 / 2.0; // Cutoff frequency.
    // Why cutoff is important can be found in the wiki link below

    // Filtering parameters
    let rc = 1.0 / (cutoff * 2.0 * std::f32::consts::PI);
    let dt = 1.0 / (input_rate as f32);
    let alpha = dt / (rc + dt);

    let mut output = Vec::with_capacity(input.len() / factor);
    let mut previous = input[0];
    let mut index = 0;

    while index < input.len() {
        let filtered_sample = previous + alpha * (input[index] - previous);
        if index % factor == 0 { // Keeping only every factor-th sample
            output.push(filtered_sample);
        }
        previous = filtered_sample;
        index += 1;
    }

    output
}
```

Granted it looks a bit complicated but in reality what it's doing is applying a low pass filter and keeping every `factor`-th sample. More details are in this [wikipedia article](https://en.wikipedia.org/wiki/Sample-rate_conversion). It's important to note if we'd just keep every nt-h sample and nothing else the pitch would get messed up. That's what the filter is for. I found that out the hard way.

And lastly we send the data into the pipeline via `mic_tx`. If this fails it means we'll never be able to send anything again so we shutdown Jarvis as well.

## Putting it all together

We've looked at the various bits and how they work and what we need so it's time to put them together and make it work. Here's the final `microphone_listener.rs` with some explainers in the comments for the missing bits.

```rust
use std::{sync::{mpsc::Sender, Arc}, time::Duration};

use anyhow::{Ok, Result};
use cpal::{traits::{HostTrait, StreamTrait}, InputCallbackInfo, SampleRate, StreamConfig};
use rodio::DeviceTrait;

use crate::{core::{constants::{AUDIO_SAMPLE_RATE, MIC_SAMPLE_RATE}, jarvis_signals::JarvisSignals}, errors::jarvis_error::JarvisError};

// Config
const INPUT_STREAM_CONFIG: StreamConfig = StreamConfig {
    channels: 1,
    sample_rate: SampleRate(MIC_SAMPLE_RATE), // I've moved the sample rate to core/constants.rs
    buffer_size: cpal::BufferSize::Default
};

pub fn main(signals: Arc<JarvisSignals>, microphone_tx: Sender<Vec<f32>>) -> Result<()> {

    // Data callback bit
    let data_signals = signals.clone();
    let data_callback = move |data: &[f32], _: &InputCallbackInfo| {
        if data_signals.is_speaker_active() || data_signals.is_shutdown() {
            return;
        }

        let resampled = resample_audio(
            data,
            MIC_SAMPLE_RATE as usize,
            AUDIO_SAMPLE_RATE
        );
        
        if let Err(e) = microphone_tx.send(resampled) {
            data_signals.set_shutdown(Some(e.into())); 
        }
    };

    // Error callback bit
    let error_signals = signals.clone();
    let error_callback = move |error: cpal::StreamError| {
        match error {
            cpal::StreamError::DeviceNotAvailable => error_signals.set_shutdown(Some(error.into())),
            _ => eprintln!("Capture stream error {:?}", error)
        };
    };

    // Creating the stream
    let stream = cpal::default_host()
        .default_input_device()
        .ok_or(JarvisError::no_mic())?
        .build_input_stream(
            &INPUT_STREAM_CONFIG,
            data_callback,
            error_callback,
            None)?;

    // Starting the capture. This activates the data_callback
    stream.play()?;

    // Since our processing in the data callback is asynchronous
    // we need to wait ie. keep this thread alive until it's time
    // to shutdown. So we just periodically check 10 times per second
    // if it's time to stop
    while !signals.is_shutdown() {
        std::thread::sleep(Duration::from_millis(100));
    }

    // Attempt to gracefully stop the stream
    stream.pause()?;

    Ok(()) //We're done!
}

// Also covered. We could potentially move it out to `core`
// but this function is only ever used here
fn resample_audio(input: &[f32], input_rate: usize, output_rate: usize) -> Vec<f32> {
    let factor = input_rate / output_rate;
    let cutoff = output_rate as f32 / 2.0;

    let rc = 1.0 / (cutoff * 2.0 * std::f32::consts::PI);
    let dt = 1.0 / (input_rate as f32);
    let alpha = dt / (rc + dt);

    let mut output = Vec::with_capacity(input.len() / factor);
    let mut previous = input[0];
    let mut index = 0;

    while index < input.len() {
        let filtered_sample = previous + alpha * (input[index] - previous);
        if index % factor == 0 {
            output.push(filtered_sample);
        }
        previous = filtered_sample;
        index += 1;
    }

    output
}
```

Nothing in the code above should look surprising. But it's nice to see it all fit together so that we can understand the big picture.

And the last piece is how the `main.rs` for microphone looks like:

```rust
let (mic_tx, mic_rx) = channel::<Vec<f32>>(); // the channel
let mic_signals = signals.clone(); // signals for the actual `main`
let mic_shutdown_signals = signals.clone(); // Signals for any unrecoverable error
thread_pool.spawn_blocking(move || {
    processing::microphone_listener::main(mic_signals, mic_tx)
        .map_err(|e| mic_shutdown_signals.set_shutdown(Some(e))) // In case `main` returns an error we shut Jarvis down
        .ok();

    // Last
    println!("Microphone listener shutting down");
});
```

**Good job!** We've got our mic input working. If we set a listener to the `mic_rx` we'll see an endless stream of floats coming through! More on that in the next post. See you soon.