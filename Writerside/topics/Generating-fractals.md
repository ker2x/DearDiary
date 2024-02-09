# Generating fractals

It's an old idea. I work on it from time to time but the results are not great.
I usually run it in google colab, sucking up all the free FLOPS i can get out of their infinite money pocket.

## notebook

{ignore-vars=true}
```Python
import tensorflow as tf
import matplotlib.pyplot as plt
import numpy as np
import pandas as pd
import datetime
from google.colab import drive

print(tf.__version__)

# Generate a learning set
class MandelbrotDataSet:
def __init__(self, size=256, max_depth=100, xmin=-2.0, xmax=0.7, ymin=-1.3, ymax=1.3):
self.data = {}
self.x = tf.random.uniform((size,),xmin,xmax,tf.float32)
self.y = tf.random.uniform((size,),ymin,ymax,tf.float32)
self.outputs = self.mandel(x=self.x, y=self.y,max_depth=max_depth)
self.data = tf.stack([self.x, self.y], axis=1)

    @staticmethod
    def mandel(x, y, max_depth):
        zx, zy = x,y
        for n in range(1, max_depth):
            zx, zy = zx*zx - zy*zy + x, 2*zx*zy + y
        return tf.less(zx*zx+zy*zy, 4.0)

class MandelbrotPointGenerator(tf.keras.utils.Sequence):
def __init__(self):
self.x = {}
self.y = {}

    def __len__(self):
        return 128

    def __getitem__(self, epoch):
        self.x = tf.random.uniform((4096,),-2.0,0.7,tf.float32)
        self.y = tf.random.uniform((4096,),-1.3,1.3,tf.float32)
        return tf.stack([self.x, self.y], axis=1), self.mandel(x=self.x, y=self.y,max_depth=100)

    @staticmethod
    def mandel(x, y, max_depth):
        zx, zy = x,y
        for n in range(1, max_depth):
            zx, zy = zx*zx - zy*zy + x, 2*zx*zy + y
        return tf.less(zx*zx+zy*zy, 4.0)

#trainingData = MandelbrotDataSet(10_000)
model = tf.keras.Sequential([
tf.keras.Input(shape=(2,)),
tf.keras.layers.Dense(1024,activation="LeakyReLU"),
tf.keras.layers.Dense(1024,activation="LeakyReLU"),
tf.keras.layers.Dense(512,activation="LeakyReLU"),
tf.keras.layers.Dense(256,activation="LeakyReLU"),
tf.keras.layers.Dense(128,activation="LeakyReLU"),
tf.keras.layers.Dense(64,activation="LeakyReLU"),
tf.keras.layers.Dense(32,activation="LeakyReLU"),
tf.keras.layers.Dense(8,activation="LeakyReLU"),
tf.keras.layers.Dense(8,activation="LeakyReLU"),
tf.keras.layers.Dense(2,activation="LeakyReLU"),
tf.keras.layers.Dense(1,activation="sigmoid")
])
#model.compile(loss=tf.keras.losses.MeanSquaredError(), optimizer=tf.keras.optimizers.Adam(learning_rate=0.001),metrics=["mae"])
model.compile(optimizer="adam", loss="binary_crossentropy", metrics=["accuracy"])
log_dir = "logs/fit/" + datetime.datetime.now().strftime("%Y%m%d-%H%M%S")
tensorboard_callback = tf.keras.callbacks.TensorBoard(log_dir=log_dir, histogram_freq=1)

sequence = MandelbrotPointGenerator()
history = model.fit(sequence,epochs=20)
np.set_printoptions(precision=3, suppress=True)

hist = pd.DataFrame(history.history)
hist['epoch'] = history.epoch
hist.tail()
def plot_loss(history):
plt.plot(history.history['accuracy'], label='accuracy')
plt.xlabel('Epoch')
plt.ylabel('accuracy')
plt.legend()
plt.grid(True)

plot_loss(history)

trainingData = MandelbrotDataSet(100_000)
predictions = model.predict(trainingData.data)
print("predictions shape:", predictions.shape)

plt.scatter(trainingData.x, trainingData.y, s=1, c=predictions)

model.save('my_model')
new_model = tf.keras.models.load_model('my_model')

# Check its architecture
new_model.summary()

trainingData = MandelbrotDataSet(size=100_000, max_depth=100, xmin=-2.0, xmax=-1.0, ymin=-0.3, ymax=0.3)
new_predictions = new_model.predict(trainingData.data)
print("predictions shape:", predictions.shape)

plt.scatter(trainingData.x, trainingData.y, s=1, c=new_predictions)

# Mount Google Drive
#drive.mount('/content/drive')

# Create a folder in the root directory

#model.save('/content/drive/My Drive/My Model/dynamic_tensorbroat_3')
```

## Retraining a saved model

```
import tensorflow as tf
import matplotlib.pyplot as plt
import numpy as np
import pandas as pd
import datetime
from google.colab import drive

print(tf.__version__)

drive.mount('/content/drive', force_remount=False)
new_model = tf.keras.models.load_model('/content/drive/My Drive/My Model/dynamic_tensorbroat_3')

# Check its architecture
new_model.summary()

class MandelbrotPointGenerator(tf.keras.utils.Sequence):
    def __init__(self):
        self.x = {}
        self.y = {}

    def __len__(self):
        return 512

    def __getitem__(self, epoch):
        self.x = tf.random.uniform((512,),-2.0,0.7,tf.float32)
        self.y = tf.random.uniform((512,),-1.3,1.3,tf.float32)
        return tf.stack([self.x, self.y], axis=1), self.mandel(x=self.x, y=self.y,max_depth=100)

    @staticmethod
    def mandel(x, y, max_depth):
        zx, zy = x,y
        for n in range(1, max_depth):
            zx, zy = zx*zx - zy*zy + x, 2*zx*zy + y
        return tf.less(zx*zx+zy*zy, 4.0)

sequence = MandelbrotPointGenerator()
new_model.fit(sequence, epochs=200)
new_model.save("/content/drive/My Drive/My Model/dynamic_tensorbroat_3")

# Generate a learning set
class MandelbrotDataSet:
    def __init__(self, size=256, max_depth=100, xmin=-2.0, xmax=0.7, ymin=-1.3, ymax=1.3):
        self.data = {}
        self.x = tf.random.uniform((size,),xmin,xmax,tf.float32)
        self.y = tf.random.uniform((size,),ymin,ymax,tf.float32)
        self.outputs = self.mandel(x=self.x, y=self.y,max_depth=max_depth)
        self.data = tf.stack([self.x, self.y], axis=1)

    @staticmethod
    def mandel(x, y, max_depth):
        zx, zy = x,y
        for n in range(1, max_depth):
            zx, zy = zx*zx - zy*zy + x, 2*zx*zy + y
        return tf.less(zx*zx+zy*zy, 4.0)

trainingData = MandelbrotDataSet(size=200_000, max_depth=100, xmin=-2.0, xmax=0.7, ymin=-1.3, ymax=1.3)
new_predictions = new_model.predict(trainingData.data)
plt.scatter(trainingData.x, trainingData.y, s=1, c=new_predictions)

trainingData = MandelbrotDataSet(size=200_000, max_depth=100, xmin=-2.0, xmax=-1.0, ymin=-0.3, ymax=0.3)
new_predictions = new_model.predict(trainingData.data)
plt.scatter(trainingData.x, trainingData.y, s=1, c=new_predictions)
```