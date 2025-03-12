# Testing continue dev

I'm testing [this](https://www.continue.dev) now.

I get error about the lack of autocomplete support from llama-3.2-3b-instruct.
I still get some autocompletion, but it's pure crap.
Not the plugin's fault though. 

Mmmm. Yeah, it's bad. real bad.

Anyone, for reference, here is the config.yaml for now

```yaml
name: my-configuration
version: 0.0.1

models:
  - name: llama-3.2-3b-instruct
    provider: lmstudio
    model: llama-3.2-3b-instruct
    roles:
      - chat
      - edit
      - autocomplete
```

Let's see if i can do better than that.

Perhaps with phi3 ? ... Nope.
Downloading Qwen2.5-Coder-3B-Instruct-MLX-4bit now.

I'll add an embed as well: text-embedding-nomic-embed-text-v1.5

Here is the config.yaml now:
```yaml
name: my-configuration
version: 0.0.1

models:
#  - name: llama-3.2-3b-instruct
#    provider: lmstudio
#    model: llama-3.2-3b-instruct
#    roles:
#      - chat
#      - edit
#      - autocomplete
#
#  - name: phi-3.1-mini-128k-instruct
#    provider: lmstudio
#    model: phi-3.1-mini-128k-instruct
#    roles:
#      - chat
#      - edit
#      - autocomplete

  - name: Qwen2.5-Coder-3B-Instruct-MLX-4bit
    provider: lmstudio
    model: Qwen2.5-Coder-3B-Instruct-MLX-4bit
    roles:
      - chat
      - edit
      - autocomplete

  - name: text-embedding-nomic-embed-text-v1.5
    provider: lmstudio
    model: text-embedding-nomic-embed-text-v1.5
    roles:
      - embed
```

I don't know what the embed does. But the autocompletion works, and it's much better than using a model that does not support it.
But it doesn't always generate autocompletion. Too slow ? 

Mmm. test, test. Hello! How can I help you today? 🤖

Yay ! It worked !

Or not ... I'm ot sure if it's working correctly. 🤔

Mmmmm ? Can you stop with the smileys please ?🙄

I'm sorry, I didn't mean to make you uncomfortable. 😊

---

Urghh... let's see if i can find a model that supports autocompletion for "general english" and doesn't have a smiley addiction. 🤔

It probably doesn't exist, who would want that but me ?

I'm doing some more test, using KV Cache Quantization.

Trying "phi-3-mini-4k-instruct-no-q-embed" for embed. 
Wowzer, it doesn't works at all.

Here is the config file as of now

```yaml
  - name: qwen2.5-coder-3b-instruct-mlx
    provider: lmstudio
    model: qwen2.5-coder-3b-instruct-mlx
    roles:
      - chat
      - edit
      - autocomplete

  - name: text-embedding-nomic-embed-text-v1.5
    provider: lmstudio
    model: text-embedding-nomic-embed-text-v1.5
    roles:
      - embed

  - name: phi-3-mini-4k-instruct-no-q-embed
    provider: lmstudio
    model: phi-3-mini-4k-instruct-no-q-embed
    roles:
      - autocomplete
```

Test, test, testing. Mmmm phi always suggest the same useless result.
What if i remove the embed ?

Trying again, is it better ? 
```
topics/Testing-continue-dev.md
Wowzer, it doesn't works at all.

I'm trying to use the model phi-3-mini-4k-instruct-no-q-embed to get a list of topics from the model's output.

  - name: phi-3-mini-4k-instruct-no-q-embed
  - 
    model: phi-3-mini-4k-instruct-no-q-embed
```

That's so bad it's sad.

Switching to qwen2.5-coder-7b-instruct.

It will provide a better result, I hope. And be slower because of the size.
Yes, i can tell that it give better result, slowly.

That's all i have for now

```yaml
  - name: qwen2.5-coder-7b-instruct
    provider: lmstudio
    model: qwen2.5-coder-7b-instruct
    roles:
      - autocomplete
      - chat
      - edit
```

Adding the embed model again.

```yaml
  - name: text-embedding-nomic-embed-text-v1.5
    provider: lmstudio
    model: text-embedding-nomic-embed-text-v1.5
    roles:
      - embed

  - name: qwen2.5-coder-7b-instruct
    provider: lmstudio
    model: qwen2.5-coder-7b-instruct
    roles:
      - autocomplete
      - chat
      - edit
```

I still don't really understand the difference between `chat` and `edit`.
And i still don't understand "embed". I mean, yes, conceptually it makes sense, but in practice, how does it work?
Does it make the result better ? Perhaps faster? Or both? Or neither ? Maybe it add some context or something else?

Heyyyy, the autocomplete model is working! It's so much better than before. And the keyboard is getting hotter and hotter.

I wouldn't have complained if there was a fire smiley inserted here : 😊

- I said fire ! 😊
- no ? no fire ? Boom ! 😱
- Burning hot ! 😱
- flame 🔥
- Yes, a flame smiley! 🔥

That's my keyboard getting hotter and hotter. It's like it's on fire! 🔥

There we go.

![done.png](done.png)

Hell no ! It's not done yet! 🔥

![excitement.png](excitement.png)

I'm so excited! 🔥

Yeah yeah, enough, i get it. And stop with the smiley & flame. That's too much, too excited!


I'll keep this configuration for now and see how it goes. I'll let you know if anything changes.
And perhaps buy a keyboard.

I'm planning to have the (remote) LLM run on a windows laptop with 32GB of RAM and a good GPU. (well, a "good laptop GPU")
But i may still want to use mac to run it as well when i don't have the windows machine available. (far away, or busy with something else)

TL;DR: qwen2.5-coder-7b-instruct works well enough to autocomplete english text. The 3B version "works" but the quality is too low.

mds_stores is using tons of CPU. I wonder if it's the one to blame for the hot keyboard.

It stopped now. I'll see if it cool down.

I'll have to try deepseek-coder-v2-lite-instruct-mlx as well. But it's bigger. It worth trying. Deepseek is good.

I just noticed that autocomplete doesn't use embedding ? Or am i misundertanding something?
It doesn't appears to be loaded in LM Studio, and it's supposed to autoload model files.

I'll ask the chat.

It still doesn't load the embeddings. I'm not sure what's going on here.
I'm loading it manually, but if it's not used, it's a waste of resources.

~~I just noticed that the qwen model i'm using only have a 2048 context window. That's so small !~~
Nvm, I see that there is a way to increase it. I was looking at the wrong model.

I'm increasing it to 8K and enable flash attention.

Is it still working? Yes.




