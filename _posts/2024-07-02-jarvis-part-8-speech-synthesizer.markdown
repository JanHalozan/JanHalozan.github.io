---
layout:     post
title:      "Jarvis miniseries part VII: Speech synthesis based on generated feedback"
subtitle:   "The journey of our AI-powered home assistant in Rust comes to an end with a text to speech synthesizer"
date:       2024-07-01 20:00:00
author:     "Jan"
header-img: "img/post-jarvis/bg_synthesizer.jpg"
---

# Where we left off

In the last post we took the `Intent` structure and prepared a feedback response to be conveyed back to the user in a text format. We used a templating system for the command branch and an entertaining AI approach for the question answering branch.

## Goal of this post

<a href="/img/post-jarvis/jarvis_processing_pipeline_speech.png">
![Image of the processing pipeline](/img/post-jarvis/jarvis_processing_pipeline_speech.png)
</a>

And just like that we're at the end of the processing pipeline! This is the last post of the series. The only thing left to do is to take the generated text from Part VII and convert it back into audio so that it can be played through the speakers. Let's get to it!

## Audio playback

We used [cpal](https://github.com/RustAudio/cpal) as the microphone input library. We could use it again but we'd have to handle some WAV specifics ourselves. Fortunately the same RustAudio group has another helpful crate we can use: [rodio](https://github.com/RustAudio/rodio). It is a layer on top of cpal with some extra functionality. Instead of going into each separate class we can check a snippet from their examples folder.

```rust
use std::io::BufReader;

fn main() {
    let (_stream, handle) = rodio::OutputStream::try_default().unwrap();
    let sink = rodio::Sink::try_new(&handle).unwrap();

    let file = std::fs::File::open("assets/music.wav").unwrap();
    sink.append(rodio::Decoder::new(BufReader::new(file)).unwrap());

    sink.sleep_until_end();
}
```

And comparing this to a simple [beep in cpal](https://github.com/RustAudio/cpal/blob/master/examples/beep.rs) it looks a lot easier to wrap our head around. With this example we can prepare most of our `speech_synthesizer.rs`.

## Speech synthesizer

At long last we'll have a `main` that takes only a `Receiver` and doesn't output anything to the next stage. The train has arrived at the station. Let's prepare the structure of the synthesizer.

```rust
use std::{io::Cursor, sync::{mpsc::Receiver, Arc}};

use anyhow::{Ok, Result};
use bytes::Bytes;
use rodio::{Decoder, OutputStream, Sink};

use crate::core::jarvis_signals::JarvisSignals;

pub fn main(signals: Arc<JarvisSignals>, feedback_rx: Receiver<String>) -> Result<()> {
    let (_stream, stream_handle) = OutputStream::try_default()?;
    let sink = Sink::try_new(&stream_handle)?;

    while !signals.is_shutdown() {
        let text = match feedback_rx.recv() {
            std::result::Result::Ok(str) => str,
            Err(_) => break
        };

        // The last missing piece is `get_audio_data`
        let audio_data = match get_audio_data(text) {
            Ok(data) => data,
            // because it can fail we want to have a fallback audio block
            Err(_) => read_fallback_feedback()
        };
        let source = Decoder::new_wav(
            Cursor::new(audio_data)
        )?;

        // While speaker output is happening we set the speaker
        // to be active so that any microphone input is discarded
        signals.set_speaker_active(true);
        sink.append(source);
        sink.sleep_until_end();
        signals.set_speaker_active(false);
    }

    Ok(())
}

fn get_audio_data(text: String) -> Result<Bytes> {
    // Generate WAV somehow    
}
```

Taking a look there's not much new here. We establish a stream first. 
_One side note is that even though `_stream` variable isn't used anywhere replacing it with `_` will cause the app to crash. That's due to the fact that `stream` needs to be alive in order for `stream_handle` to work. If it's assigned to `_stream` (the underscore so that compiler doesn't complain about an unused variable) it is dropped only after it goes out of scope opposed to immediately in the other scenario._

We then just obtain the audio data from `get_audio_data`.

## Getting audio data

I had quite a bit of back and forth on this step mainly due to trying to find the right library for speech synthesis. I want Jarvis to run completely locally without requiring an internet connection meaning that all external services are off limits. Trying to find a good crate was equally as hard (possibly a good idea for a follow up pet project). So in the end I decided to use [coqui-ai's TTS](https://github.com/coqui-ai/TTS) that would be installed as a command line tool and invoked by Rust. It's a very flexible library with support for different voices and languages meaning it wouldn't be to hard to  add some intermediate processing steps and port Jarvis to a different language.

That having been said let's get Jarvis to start speaking back.

We can follow the installation instructions for TTS on the repository readme. If you have the right python versions etc it's as simple as `pip install tts`. Unfortunately I didn't so it took me a bit of configurations to get the right python and pip versions but I did manage to get it working.

We can find a lot of configuration options for the `tts` command by running `tts --help`. The two we'll use are:
- `--text "text to speak"` this one should be pretty self explanatory.
- `--pipe_out` pipes the raw audio data directly into `stdout`. Instead of having an temporary file that we read we can just capture the `stdout` stream and pass it along as raw wav data.

There's a lot more and you can play around with them as you'd like. It's interesting to see how many different models and voices are available.

### Implementing `get_audio_data`

Running external commands in Rust is pretty straighforward as the standard library provides `std::process::Command`.

```rust
fn get_audio_data(text: String) -> Result<Vec<u8>> {
    // Create the command
    let mut child = std::process::Command::new("tts")
        // Pass the arguments, you can add others for a different voice
        .args(&["--text", text.trim(), "--pipe_out"])
        .stdout(Stdio::piped())
        .spawn()?;

    // Input buffer
    let mut audio_data = Vec::new();
    child
        .stdout
        .as_mut()
        .context("Could not get TTS stdout.")?
        // Read the stdout into the audi_data buffer
        .read_to_end(&mut audio_data)?;

    // Wait until tts completes
    child.wait()?;

    // Return the buffer
    Ok(audio_data)
}
```

With that we've got the barebones implementation of synthesizing speech through an external command. 

## Wrapping up!

This concludes the last bits of the last stage of the processing pipeline. If we build and run Jarvis now it will interpret our speech, flesh out the command, execute it (not really) and provide vocal feedback confirmation!

I will likely add some follow up posts for running Jarvis on different devices and performing tasks. I might refactor some code and update the articles as well.

It has surely been a journey and I'd like to thank you massively for perserveering. If you have any questions feel free to reach out and I hope you enjoyed this journey as much as I did.

Godspeed.