# LLM on SSD

I simply have some kind of general feeling that a LLM may not absolutely need a large amount of fast memory to run.

There is a [paper about it](https://arxiv.org/abs/2312.11514) named "LLM in Flash".
I need to read it first.

> Large language models (LLMs) are central to modern natural language processing, delivering exceptional performance in various tasks. However, their substantial computational and memory requirements present challenges, especially for devices with limited DRAM capacity. This paper tackles the challenge of efficiently running LLMs that exceed the available DRAM capacity by storing the model parameters in flash memory, but bringing them on demand to DRAM. Our method involves constructing an inference cost model that takes into account the characteristics of flash memory, guiding us to optimize in two critical areas: reducing the volume of data transferred from flash and reading data in larger, more contiguous chunks. Within this hardware-informed framework, we introduce two principal techniques. First, "windowing" strategically reduces data transfer by reusing previously activated neurons, and second, "row-column bundling", tailored to the sequential data access strengths of flash memory, increases the size of data chunks read from flash memory. These methods collectively enable running models up to twice the size of the available DRAM, with a 4-5x and 20-25x increase in inference speed compared to naive loading approaches in CPU and GPU, respectively. Our integration of sparsity awareness, context-adaptive loading, and a hardware-oriented design paves the way for effective inference of LLMs on devices with limited memory.

To be honest, I expected more than  "up to twice the size of the available DRAM".
What about 10x ? 100x ? What the point of using a LLM if you can't use a large one ?

The 4-5x and 20-25x increase in inference speed is interesting though. But not the point.

## The cost of running a LLM

LLM can't be forever limited by memory and can't always be on large, expensive, cloud servers.
The public don't understand the insane ecological and economical cost of running a LLM.
It really is a shame to use a LLM for simple requests like "what is the weather today ?" or "what is the capital of France ?".

And yes, I'm aware of how ironic it is to use a LLM to write about the ecological cost of using a LLM. 
I'm not sure if it's funny or sad.

## The future

I don't understand the paper so that's it for now until something else come up.
The paper is published by Apple, if there is value in this research it will appear on some Apple product in the future.