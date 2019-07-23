---
published: true
title: A Python Data Storage Trick
layout: post
---
Here's a trick I just came up with. When messing about with training neural networks, I can sometimes find myself generating and manipulating hundreds of thousands of bitmap images, which are essentially binary matrices. With font data you can generate these on the fly and feed them straight into your network, but it turns out that takes a lot of RAM, and it's more efficient to pre-compute them.

So, how do you efficiently store and handle quarter of a million bitmap files? You could, of course, save them as images in a directory and have a generator just pull out a file at a time, but that's horrible for a couple of reasons: first, it's inefficient because your matrices are generally going to be sparse (lots of zero bits), and second, directories with huge numbers of files in can cause your file system to choke.

Here's an alternative approach:

```Python
ix = 0
images = {}
while making_stuff:
	ix = ix + 1
    images["ix-%06i" % ix] = make_an_image()
np.savez_compressed("training", **images)

# Later
t_inputs = np.load("training.npz")
t_outputs = np.load("out-training.npz")["outputs"]
def generator:
  t_indices = []
  while True:
    if len(t_indices) == 0:
      t_indices = list(range(0,len(t_outputs)))
      random.shuffle(t_indices)
    ix = t_indices.pop()
    input_x = t_inputs["ix-%06i" % ix]
    yield input_x, t_outputs[ix]
```

A numpy NPZ file is a zip file of numpy objects. By adding each image as a separate object in the file, you end up with something like this:

```
$ unzip -l training.npz | head
Archive:  -training.npz
  Length      Date    Time    Name
---------  ---------- -----   ----
    88128  01-01-1980 00:00   ix-000000.npy
    88128  01-01-1980 00:00   ix-000001.npy
    ...
```

This means: (a) you get excellent compression of your bitmap images - for me, two gig of raw data compresses down to about 300M on disk, (b) you get a single to pass around rather than an unwieldy monster directory, and you don't have to worry about sharding, and (c) you get fast random access to each individual image with low RAM requirement (don't have to read the whole dataset into memory).