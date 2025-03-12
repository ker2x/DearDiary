# The fun with continue continue

What a terrible title, i know. But it's true!
And i have, like, zero shame for it.
Am i getting old enough for a dad joke? I surely am!

Even jetbrains complain about a possible typo for repeating words.

Oh well... 

> Mmmm LLM request are queuing up, look like I managed to crash something with my sense of humor.
> Amazeball...

sigh ... I wonder if 

![career-end.png](career-end.png)

Wow ! Too much drama ! I even forgot what i was wondering about.

Nvm, i remember, temperature setting. 

Trying a new config:

```yaml
  - name: qwen2.5-coder-7b-instruct
    provider: lmstudio
    model: qwen2.5-coder-7b-instruct
    defaultCompletionOptions:
      contextLength: 8000
      temperature: 0.9
    roles:
      - autocomplete
      - chat
      - edit
```

I'll add a "draft model" as well (Qwen2.5 Coder 0.5B Instruct GGUF), this is a LM Studio setting in "Speculative Decoding".
I'll ask gemini what it's all about.

## What is speculative decoding and draft model in LLM ?

When discussing Large Language Models (LLMs), "speculative decoding" and "draft model" refer to techniques designed to accelerate the process of generating text. Here's a breakdown:

**Speculative Decoding:**

* **The Problem:**
    * LLMs, while powerful, can be slow when generating text because they typically produce one token (a word or part of a word) at a time. This sequential process leads to latency.
* **The Solution:**
    * Speculative decoding aims to speed up this process by having a smaller, faster "draft" model propose a sequence of potential tokens.
    * Then, the main, more accurate (but slower) LLM verifies these proposed tokens.
    * If the draft model's suggestions are correct, they're accepted, significantly speeding up generation. If they are incorrect, the larger model corrects them.
    * Essentially it is like having a fast first draft produced, and then having a very good editor check and correct that draft.
* **Key Idea:**
    * It allows for parallel processing, where the main LLM can verify multiple tokens at once, rather than generating them one by one.

**Draft Model:**

* **Role:**
    * The draft model is a smaller, less complex LLM that's used to generate the "draft" of potential tokens in speculative decoding.
* **Characteristics:**
    * It's designed to be fast, even if it's less accurate than the main LLM.
    * Its purpose is to provide a reasonable guess of the next tokens, which the main LLM can then validate.
    * The draft model is trained to be able to give a reasonable next sequence of tokens, even if it is not always perfectly accurate.
* **Importance:**
    * The efficiency of speculative decoding heavily depends on the draft model's speed and its ability to generate reasonably accurate suggestions.

In summary, speculative decoding is a technique that leverages a faster draft model to accelerate LLM inference, and the draft model itself is the auxiliary, faster model that produces the initial token suggestions.

## The fun with continue continue continue

Yes, obviously i had to do this. Anyway ...
The speculative decoding thing not only make sense, but it appears to work.

I've also set temperature to such a high value that i end up never accepting the autocompletion result.
I'll dial it down to 0.5. There is a fine line between creativity and insanity.

And i don't need an insane model for this to work, just a fast one that can make good guesses.

It's also difficult to say if speculative decoding is getting in the way.
Because, of course, i had to make 2 changes at once. Which means i'm not sure if it's the speculative decoding or the temperature that's causing issue.

But the lower temperature dialed down the madness, and made it more reasonable.

I'll disable the speculative decoding and see what happens.

(by the way, i'm accepeing most autocompletion, that's a good sign.)

There we go, no speculative decoding. It works! Faster ? Slower ? Better? I don't know yet.

I can't spot any difference at all. I feel like it's faster, which is counter intuitive.

Gemini to the rescue again

## Is it really faster ?

Yes, speculative decoding can significantly accelerate LLM text generation, but its effectiveness depends on several factors. Here's a breakdown:

**How it achieves speedups:**

* **Parallel Processing:**
    * The core idea is to shift from a strictly sequential token generation process to a more parallel one. By having a draft model propose multiple tokens, the main LLM can verify them in a batch, reducing the overall latency.
* **Leveraging Smaller Models:**
    * Smaller models are inherently faster. By using a draft model for the "easy" parts of the generation, the system avoids the computational overhead of running the large LLM for every single token.

**Factors influencing speed:**

* **Draft Model Accuracy:**
    * The more accurate the draft model, the higher the acceptance rate of its proposed tokens. If the main LLM frequently rejects the draft's suggestions, the efficiency gains diminish.
* **Draft Model Size:**
    * A balance must be struck. A very small draft model might be too inaccurate, while a draft model that's too large might negate the speed benefits.
* **Hardware and Implementation:**
    * The specific hardware and software implementation play a crucial role. Efficient parallel processing and optimized memory management are essential for maximizing the speedup.
* **Target model size:**
    * It has been seen that the speed up ratio increases as the target model size increases.

**General Observations:**

* Studies and implementations have shown that speculative decoding can achieve speedups of 2x to 3x or even more in certain scenarios.
* The technique is particularly beneficial for large, computationally intensive LLMs.

In essence, speculative decoding is a valuable optimization that can make LLMs more practical for real-time applications.

## Mmmmm

From my understanding i'm supposed to gain from parallel processing.
But the base model is kind of maxing out my M3 already so there might not be much to gain, or worse. I could be losing efficiency due to speculative decoding. 

Doing sequential work when the gain comes from parallelism.
It's just additional more work that doesn't really help.

I enable it again.

Yep, this is what i'm talking about. It's a waste of time and resources if the base model is already maxing out my M3.

But... While it's not faster, i feel like i'm getting better results with speculative decoding enabled.

There is this thing too:

> The more accurate the draft model, the higher the acceptance rate of its proposed tokens. If the main LLM frequently rejects the draft's suggestions, the efficiency gains diminish.

A 0.5B model, specialized in code, to generate english. I might be asking for something that is outside its capabilities.

LM Studio is also recommending "DeepSeek R1 Distill Qwen 1.5B GGUF" but i'm unable to select it.

> Studies and implementations have shown that speculative decoding can achieve speedups of 2x to 3x or even more in certain scenarios.

Nope. Not even close. More like 2% than 2x.

Disabling it again.

- Time to read [some paper](https://arxiv.org/pdf/2401.07851v2) about it.
- And [this one](https://arxiv.org/pdf/2502.04557v1) as well

I have yet to read them. But i realize that it could be more useful on much slower model than the fast qwen 7B i'm currently using.

The problem is that i don't have that much RAM in the first place to load a "much slower" model.

In the first paper, it state "the main latency bottleneck arising from memory reads/writes of LLM parameters rather than arithmetic computations"

Ha ! This might be true on GPU, but i'm on M3. With a possibly completely different architecture and hardware design, this might not hold true at all.

> Does this hold true on an apple silicon, which have a different architecture and might have different kind of bottleneck ?

It's true that Apple silicon's unique architecture presents a different landscape for LLM performance. Here's how speculative decoding fares in that context:

**Key Considerations for Apple Silicon:**

* **Unified Memory Architecture:**
    * Apple silicon's unified memory architecture, where CPU and GPU share the same memory, can potentially reduce memory transfer bottlenecks, which are often a limiting factor in LLM inference. This can be advantageous for speculative decoding, as it involves frequent data exchanges between the draft and main models.
* **Neural Engine:**
    * Apple's Neural Engine is specifically designed for machine learning tasks, and optimizations that leverage this hardware can yield significant performance gains.
* **MLX Framework:**
    * Apple has developed the MLX framework, which is tailored for Apple silicon and aims to optimize machine learning workloads. This framework is crucial for maximizing the efficiency of LLM inference on these chips.
* **Specific Optimizations:**
    * Because of the different architecture, optimizations that work very well on Nvidia GPUs, may not translate perfectly to Apple Silicon. There is a need for optimizations that are specific to the Apple Silicon architecture.

**Speculative Decoding on Apple Silicon:**

* Research from Apple itself indicates that speculative decoding can indeed provide speedups on Apple silicon.
* Specifically, work like "Recurrent Drafter" shows that by utilizing techniques optimized for Apple silicon, significant accelerations are possible.
* However, the degree of speedup can vary depending on:
    * The specific Apple silicon chip (e.g., M1, M2, M3, and their Pro/Max/Ultra variants).
    * The size and architecture of the LLMs involved.
    * The efficiency of the software implementation (e.g., using MLX effectively).
* It has been seen that by using the MLX framework, and by optimizing for the Apple silicon architecture, that good speed up results can be achieved.

**In essence:**

* While the underlying principles of speculative decoding remain the same, achieving optimal performance on Apple silicon requires tailored optimizations that leverage the chip's unique capabilities.
* Evidence shows that with proper implementation, speculative decoding can indeed provide valuable speedups on Apple silicon.

## Let's test with deepseek-r1-distill-qwen-7b

And adding "DeepSeek R1 Distill Qwen 1.5B" as the draft model.

I'm clearly ... clearly... hello ? autocompletion please ? Wow... slow ! It's even queuing now.

NOPE !

Forget it, unusable. Even with speculative decoding. Sigh (again).
Well that's a reasoning model.<think>

Unfit for autocompletion.

Before killing it, let's try some chat.

## I'm testing it directly in LM Studio so that i can do more test.

### With speculative decoding enabled
- 
- 13.51 tok/sec
- 831 tokens
- 0.31s to first token
- Stop reason: EOS Token Found
- Accepted 286/831 draft tokens (34.4%)

### disabled
- 
- 17.18 tok/sec
- 971 tokens
- 0.18s to first token
- Stop reason: EOS Token Found

Well well well. It's faster with speculative decoding disabled and the result is clearly better.

Now trying with qwen2.5-coder-7b-instruct.

Same kind of result. 

I will try with a remote LLM with a GPU (my other laptop). But not today.
