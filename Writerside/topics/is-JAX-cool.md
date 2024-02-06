# is JAX cool ?

Python is slow, and don't get me started on how it "call C".
Even numpy is slow (except for slow computations, then it's relatively fast).

Is JAX cool ? Does it "make Python fast" ?
Perhaps.

It can be a straight-up replacement for numpy, but faster (on GPU, TPU, or CPU fallback).

## Testing stuff

```Python

import jax.numpy as jnp
from jax import grad, jit, vmap
from jax import random
import matplotlib.pyplot as plt

%matplotlib inline 

x = jnp.arange(0,2*jnp.pi,0.1)   # start,stop,step

for j in range(20):
  y = 0
  for i in range(1,j,2):
    y += jnp.sin(x*i)/i
    plt.plot(x,y)
plt.show()

```

To be honest, it's not a good benchmark to compare with numpy.
I'm just putting it here because i always forget how to make square waves from sin (Fourier series).

But it look like numpy, so it's cool.

![squareFFT.png](squareFFT.png)

There is probably a more JNX-ish wat to do it, but how ? Ask Copilot chat ?

Copilot is still using loops. Perhaps it's how it's done in JAX.

I'll need to try some other examples. Perhaps Fractals, or Fractals.


I'll add sawtooth while i'm at it. (no need to screenshot, it's just a sawtooth)

```Python
for j in range(20):
  y = 0
  for i in range(2,j,2):
    y += jnp.sin(x*i)/i
    plt.plot(x,y)
plt.show()
```

Factorial approximation (Stirling’s formula) :

```Python
y = jnp.sqrt(2*jnp.pi*x) * jnp.power(x/jnp.e,x)
plt.plot(x,y)
plt.show()
```


## Hello mandelbrot

