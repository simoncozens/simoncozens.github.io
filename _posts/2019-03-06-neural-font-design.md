---
layout: post
title: Neural font design
published: true
---

I've been messing around recently with the idea of getting a neural network to spit out characters. It's not something I want to spend a lot of time on and develop into something serious, but it's quite a fun little project anyway.

What got me thinking about this was a chapter in Douwe Osinga's [Deep Learning Cookbook](http://shop.oreilly.com/product/0636920097471.do) about generating icons with a recurrent neural network. In the chapter, we move from the idea of trying to generate icons pixel-by-pixel to seeing an icon as a series of drawing instructions - in the book, based on scanlines. Osinga shows that by getting the network to produce drawing instructions rather than pixels you end up with icons which are much more "icon-like".

Well, a glyph in a font is just a series of drawing instructions (oncurve/offcurve, X coordinate, Y coordinate), so if we show the network a sequence of instructions from a real font, maybe it will start dreaming up its own characters. And, you know, it sort of does:

![](https://pbs.twimg.com/media/D00tyxEW0AQJ-gZ?format=png&name=small)

I'm happy to tidy up the code and make it available at some point, but here's a few observations based on what I have learnt so far:

- I based my code in Osinga's [icon drawing RNN](https://github.com/DOsinga/deep_learning_cookbook/blob/master/14.4%20Icon%20RNN.ipynb) and then heavily hacked on it.

- Encoding the X and Y coordinates is tricky. You basically want to produce three features on your output: an instruction (which could be on curve, off curve, end contour, end glyph) and two coordinates. The instruction is easy, that's a one-hot encoded vector of four bits, that you then run softmax on. Based on that I decided to bin the co-ordinates and one-hot encode them as well. It's crude and you get your co-ordinates "quantized" to the number of bins you choose, but it's simple enough. Now that I think about it on writing this down, maybe I could try leaving the last two features unencoded and seeing them as a regression problem. Huh. I'll try that and see what happens.

- Anyway, the way it's done at the moment is essentially a three-way multiclassification problem: I modified Osinga's code to use sigmoid activation and binary cross-entropy as the loss; then when I get some output, I run argmax on the first four fields to get the instruction, again on the next n(=number of bins) fields to get the encoded X coordinate and again on the rest of the fields to get the Y coordinate. Then I one-hot encode them all again, add it back into the end of the sequence and feed it back into the network:


```python
    origpreds = model.predict(x,verbose=0)
    nexttag = np.argmax(origpreds[0,0:4])
    x_coord = int(x_binner.inverse_transform(origpreds[0,4:4+bins].reshape(1, -1))[0][0])
    y_coord = int(y_binner.inverse_transform(origpreds[0,4+bins:].reshape(1, -1))[0][0])
    origpreds[0,0:4] = origpreds[0,0:4] == np.max(origpreds[0,0:4])
    origpreds[0,4:4+bins] = origpreds[0,4:4+bins] == np.max(origpreds[0,4:4+bins])
    origpreds[0,4+bins:] = origpreds[0,4+bins:] == np.max(origpreds[0,4+bins:])
    origpreds = origpreds.astype(int)

    origpreds = origpreds.reshape(1,1,2*bins+4)
    x = np.concatenate((x[0,1:,:],origpreds[0,:])).reshape(1,look_back,2*bins+4)
```

- When you have a lot of one-hot encoded bins for the coordinates, the accuracy becomes stupid. You can get 90% accuracy really quickly because most of the fields are going to be zero! So I used a custom accuracy metric to only compare the fields which have a 1 in them. I'm quite proud of that...

```python
def and_accuracy(y_true, y_pred):
  return K.sum(y_true * K.round(y_pred), axis=-1) / K.sum(y_true,axis=-1)
```

- Choosing the range of input fonts to show to the network is difficult. Choose a small number of fonts, and output just replicates input; choose a large number and the network doesn't converge or the quantized coordinates don't really relate to the metrics of the font.

- This is obviously a sequence prediction problem, but getting the right amount of lookbehind (how long a sequence of instructions the network sees at any one time) is critical. Too much and you just basically reproduce existing glyphs, which if you're training on one or two families as we mentioned above is (*cough*) starting to look like a copyright violation. But too little lookbehind and you don't produce coherent contours. Get the right amount of lookbehind and you end up with conherent contours but not necessarily existing characters - like the one above. If you're interested in creating experimental new scripts, that's quite fun. If you want to actually train a network to produce new *styles* of "real" characters, then it's not quite what you want. You could probably mitigate that by also one-hot encoding the character in your input. But I quite like the idea of getting it to dream up coherent contours of non-existing characters...
