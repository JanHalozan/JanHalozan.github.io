---
layout:     post
title:      "Jarvis miniseries part VII: Constructing feedback from Intents"
subtitle:   "It would be nice if Jarvis actually spoke back. But before it does it needs to generate feedback somehow."
date:       2024-07-01 20:00:00
author:     "Jan"
header-img: "img/post-jarvis/bg_feedback.jpg"
---

# Where we left off

Part 6 covered executing commands on a Raspberry PI and dealing with external services. Jarvis can actually turn on lights and such now. We're getting close to the end of the pipeline.

## Goal of this post

<a href="/img/post-jarvis/jarvis_processing_pipeline_feedback.png">
![Image of the processing pipeline](/img/post-jarvis/jarvis_processing_pipeline_feedback.png)
</a>

The goal of this post is to construct text feedback to let the user know what action transpired or if an error occured. We'll also handle the `Intent::Question` part here.

## Command feedback

When we say _"Jarvis turn on the light in the living room"_ and the light turns on it's still nice to hear an explicit confirmation of what Jarvis just did. We already know exactly what happened by inspecting the `Command` struct. So generating a feedback for a command is pretty straightforward.

**Disclaimer:** I tried generating feedback using AI first but all of the models I tried provided horrible feedback. I also tried GPT2 but without much success. Could have been the parameteres but eventually I gave up.

Instead of using AI for this bit I've resorted to a more manual approach using a few predetermined templates. When speaking to Siri on iOS I noticed that when performing an action its feedback was always very similar as well.

## Template approach

I'll add the code for generating feedback for a `Command` below. It makes more sense to let code do the talking since it's really not that complicated. Note that we need to add the `rand` crate here because Rust doesn't support a good mechanism of generating random numbers out the box.

```rust
fn feedback_for_command(command: &Command) -> String {
    // Stringify each part of the command
    let action = command.action.to_string();
    let subject_description = format!("the {}", command.subject.to_string());
    let location_description = format!("in the {}", command.location);

    // And randomly choose among the 5 pre-prepared templates
    // which are then filled
    match rand::thread_rng().gen_range(0..=4) {
        0 => format!("I've {} {} {}", action, subject_description, location_description),
        1 => format!("{} {} has been {}", subject_description, location_description, action),
        2 => format!("{} {} is now {}", subject_description, location_description, action),
        3 => format!("I've successfully {} {} {}", action, subject_description, location_description),
        4 => format!("Done! {} {} is now {}", subject_description, location_description, action),
        _ => unreachable!(),
    }
}
```

However in order for this function to work `CommandSubject` and `CommandAction` need to support `to_string()`. To add that we need to implement the `Display` trait.

```rust
//command_action.rs
impl Display for CommandAction {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        let str = match self {
            Self::Switch(CommandSwitchValue::On) => "turned on",
            Self::Switch(CommandSwitchValue::Off) => "turned off",
            Self::Gradient(CommandGradientValue::Min) => "closed",
            Self::Gradient(CommandGradientValue::Max) => "opened",
            Self::Gradient(CommandGradientValue::More) => "raised",
            Self::Gradient(CommandGradientValue::Less) => "lowered"
        };

        write!(f, "{}", str)
    }
}
```
```rust
//command_subject.rs
impl Display for CommandSubject {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            Self::Light => write!(f, "light"),
            Self::Teapot => write!(f, "teapot"),
            Self::WindowBlinds => write!(f, "window blinds"),
            Self::Temperature => write!(f, "temperature"),
            Self::Ventilator => write!(f, "ventilator")
        }
    }
}
```

This isn't the best approach but it is the simplest. Because if we told Jarvis to turn the temperature to the min it would reply by saying it _"closed the temperature"_. But from my actual testing it works well. To make this more sophisticated I'd either create a lot more contextualized `feedback_for_command` or have another stab at using AI to generate the confirmation text.

## Question answering

For answering questions there's again a lot of paths we could take. Because I want Jarvis to run fully locally I'll be using a pretrained AI model for answering questions. And because this part of the project is more for fun than anything else I'll use GPT2 as the answering machine. It is a far shot from the GPT3 or GPT4 models and sometimes provides accurate answers, sometimes partially accurate and sometimes absolute nonsense but it has a tendency to be fully hilarious. If you're serios about this part I'd recommend using a better model or hooking it up to an external API.

We could always get a lot more sophisticated here and have a classifier for figuring out the most popular kinds of questions like what time is it, what day is it, basic arithmetic, weather forecast etc. We could easly handle some of them.

### GPT2 to the rescue

In a similar fashion to `feedback_for_command` we'll add a `answer_for_question`.

```rust
fn answer_for_question(question: String, model: &GPT2Generator) -> String {
    let output = model.generate(Some(&[question]), None);

    println!("Question {:?}", output);
    let empty_answer = "I don't know".to_string();
    match output {
        Ok(answer) => answer
            .first()
            .map(|a| a.text.clone())
            .unwrap_or_else(|| empty_answer),
        Err(_) => empty_answer
    }
}
```

Nothing we haven't seen before. Just a different AI model and a different way of getting the result out. Another relevant bit however is the creation of the GPT generator mainly due to the configuration of it.

```rust
let config = GenerateConfig {
    model_type: rust_bert::pipelines::common::ModelType::GPT2,
    max_length: Some(30),
    min_length: 5,
    length_penalty: 20.0,
    early_stopping: true,
    do_sample: false,
    num_beams: 5,
    temperature: 0.05,
    ..Default::default()
};
let model = GPT2Generator::new(config)?;
```

These parameters all have a pretty big impact in how the final answer is generated. It's where the seriousnes and hilariousness come from. Feel free to tweak and read up on the docs on what they do.

We can now put the full `feedback_generator.rs` together

```rust
use std::sync::mpsc::{Receiver, Sender};

use anyhow::Result;
use rand::Rng;
use rust_bert::{gpt2::GPT2Generator, pipelines::generation_utils::{GenerateConfig, LanguageGenerator}};

use crate::model::{command::Command, intent::Intent};

use super::classifier::ClassificationFailureReason;

pub fn main(intent_rx: Receiver<Result<Intent, ClassificationFailureReason>>, feedback_tx: Sender<String>) -> Result<()> {
    // We'll pass the model along. An instance variable could
    // be handy here but aside from a small semantic issue
    // this is a bit easier
    let config = GenerateConfig {
        model_type: rust_bert::pipelines::common::ModelType::GPT2,
        max_length: Some(30),
        min_length: 5,
        length_penalty: 20.0,
        early_stopping: true,
        do_sample: false,
        num_beams: 5,
        temperature: 0.05,
        ..Default::default()
    };
    let model = GPT2Generator::new(config)?;

    while let Ok(result) = intent_rx.recv() { 
        // Handle the two scenarios that can happen
        let message = match result {
            Ok(intent) => feedback_for_intent(intent, &model),
            Err(error) => feedback_for_error(error)
        };
        println!("Feedback message: {}", message);
        if feedback_tx.send(message).is_err() {
            break;
        }
    }
    
    Ok(())
}

fn feedback_for_error(reason: ClassificationFailureReason) -> String {
    let str = match reason {
        ClassificationFailureReason::UnsupportedInstruction => "I don't know how to do this yet.",
        ClassificationFailureReason::UnrecognizedInstruction => "I'm not sure I recognize your instruction",
        ClassificationFailureReason::Unknown => "Sorry, something went wrong. Could you repeat that?"
    };

    str.to_string()
}

fn feedback_for_intent(intent: Intent, model: &GPT2Generator) -> String {
    match intent {
        Intent::Command(ref command) => feedback_for_command(command),
        Intent::Question(question) => answer_for_question(question, model)
    }
}

fn feedback_for_command(command: &Command) -> String {
    let action = command.action.to_string();
    let subject_description = format!("the {}", command.subject.to_string());
    let location_description = format!("in the {}", command.location);

    match rand::thread_rng().gen_range(0..=4) {
        0 => format!("I've {} {} {}", action, subject_description, location_description),
        1 => format!("{} {} has been {}", subject_description, location_description, action),
        2 => format!("{} {} is now {}", subject_description, location_description, action),
        3 => format!("I've successfully {} {} {}", action, subject_description, location_description),
        4 => format!("Done! {} {} is now {}", subject_description, location_description, action),
        _ => unreachable!(),
    }
}

fn answer_for_question(question: String, model: &GPT2Generator) -> String {
    let output = model.generate(Some(&[question]), None);

    println!("Question {:?}", output);
    let empty_answer = "I don't know".to_string();
    match output {
        Ok(answer) => answer
            .first()
            .map(|a| a.text.clone())
            .unwrap_or_else(|| empty_answer),
        Err(_) => empty_answer
    }
}
```

And with that we're nearly done with the processing pipeline. The only thing left is to add a speech synthesizer to actually perform TTS (text to speech) and play the audio via device's speakers. Here's how the code plugs into `main.rs`:

```rust
let (feedback_tx, feedback_rx) = channel::<String>();
let feedback_signals = signals.clone();
thread_pool.spawn_blocking(move || {
    processing::feedback_generator::main(executor_rx, feedback_tx)
        .map_err(|e| feedback_signals.set_shutdown(Some(e)))
        .ok();
    println!("Feedback generator shutting down");
});
```

And just like that it's onto the last post next!