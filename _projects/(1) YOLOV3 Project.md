---
name: YOLOv3
tools: [Tensorflow, Python, YOLOv3]
image: /assets/yolo/test.png
description: Implementing YOLOv3 in Tensorflow.
# external_url: https://github.com/JiaJiunn/yolo
---

# Implementing YOLOv3 in Tensorflow 

Note: you can find the project implementation [here](https://github.com/JiaJiunn/yolo).

<br />

For quite some time now, I’ve been thinking of implementing several of the more iconic object detection models — such as YOLO, Mask-RCNN, etc. — from scratch. This is in part motivated by the fact that I haven’t actually taken a machine learning course formally; most of what I know about the field is through extracurriculars, work, and the occasional medium article, giving me a scattered understanding of concepts here and there. In retrospect, I wanted to start getting a more standardized understanding of the field, and implementing these models myself would hopefully lead to a deeper understanding of their architecture details and reasoning.

Well, this week I thought I’d start off with implementing [YOLOv3](https://arxiv.org/pdf/1506.02640.pdf). Skimming through the original YOLO paper, YOLO seems fairly straightforward in comparison to other recent object recognition models. For instance, models from the R-CNN family, like R-CNN, Fast R-CNN, Faster R-CNN, and to a certain extent, Mask R-CNN, seem to approach the object detection problem by first generating potential bounding boxes, then running a classifier on these boxes. 

In retrospect, YOLO approaches our object detection problem just like a single regression problem: it shoves the image into a single convolutional network and spits out the values we want in a particular format. In contrast to R-CNN’s somewhat two-stage pipeline, YOLO is both simpler implementation-wise, and also runs faster in practice, since the network is not as complicated. Authors of the YOLOv3 paper reported that they still get performance on par with other SOTA classifiers, but with significantly faster speed:

![Performance comparison from YOLOv3 paper](/assets/yolo/model_stats.png)

That being said, let's get right down to implementing it!

## Implementation

To start off, I took a look at this config file provided by Joseph Redmon, one of the authors of the paper, who linked it on his [official website](https://pjreddie.com/darknet/yolo/). You may remember him from a few years ago, when he was massively popular on reddit for his [my little pony resume](https://pjreddie.com/static/Redmon%20Resume.pdf). Anyhow, The config file essentially specifies YOLO as a sequential order of blocks, and also provide the parameters for each block. Much like [this](https://github.com/ayooshkathuria/YOLO_v3_tutorial_from_scratch) PyTorch implementation, I thought that it’d make more sense to iterate through the config file in order to build the model, rather than manually code each block’s parameters in a huge mess of a file. This way, the code would not only be shorter and cleaner, but we can also have all the parameters  specified in one place. After parsing the config file, here’s the general idea I had in mind:

![General idea](/assets/yolo/implementation_idea.png)

Looking through the configs, the first block, `net`, seems to specify the network input (such as batch size and image size) as well as training parameters, and isn’t really specifying a block type. Starting from the second block, however, all the blocks below each specify one of five different block types. We will go through each block type below.

### Convolution block

The `convolutional` block is probably the most straightforward block. We simply parse the parameters from the config file into the Tensorflow convolution function. The only noteworthy I learned to look out for is that I find it somewhat awkward to write paddings for the input tensors; Tensorflow doesn’t support padding as one of the parameters for the in-built convolutional function, unlike PyTorch.

![Convolutional block](/assets/yolo/conv.png)

### Upsample block

The next block type is `upsampling`, which is also rather straightforward to do in Tensorflow. We simply call the resize function below, and it seems like Tensorflow by default uses bilinear interpolation for its resizing.

![Upsample block](/assets/yolo/upsample.png)

### Route block

Now, the `route` block is still straightforward, but might need some explaining. It has a parameter called layers which is either one or two numbers. The number represents the index of the layer that will be outputted by the route block. For instance, when the layer parameter has one number, -1, the route block output will the layer behind the route layer. When it has two numbers, say -1, 61, then the output will be the concatenation of the layer behind the route layer, together with the 61st layer (yes, so we need to take into account the absolute vs relative layer indices while parsing). Of course, this by itself isn’t complicated to implement; we then just set up a list `outputs` and keep track of all outputs by each layer.

![Route block](/assets/yolo/route.png)

### Shortcut block

And for the most straightforward block, the `shortcut` block is simply a skip connection as introduced in the ResNet papers. It has one parameter `from` which specifies which layer we attach the skip connection from. The activation seems to always be linear so I didn’t bother parsing it.

![Shortcut block](/assets/yolo/shortcut.png)

### YOLO block

Lastly, there are three `yolo` blocks, which require some explanation on the YOLOv3 architecture itself. 

On a high level, the way YOLO makes predictions is by first dividing up the image into equally-sized grids, where each grid is responsible for making generating `B` bounding box predictions (i.e. B objects, where B = 3 in our version of YOLOv3). The prediction for each grid has depth `B x (5 + C)`, because each prediction comes with a bounding box of format `x_center, y_center, width, height` and an objectness score, which is the predicted probability that there is indeed an object in the bounding box (4 + 1 = 5). C then represents the number of classes, and the C depth is simply the confidences for each class in that bounding box.

Now, this process of divide-into-grid-and-predict is actually done three times in our network definition, corresponding to the three [yolo] blocks. So we can think of these three `yolo` block outputs as object bounding box predictions done on three different scales, so that the network doesn’t miss out on really large or really small objects.

But this brings a catch when corresponding the bounding box `xywh` predictions to actual locations in the image. The `xywh` don’t actually correspond to the actual sizes in the original image, since we downsample the images at different scales. In this case, YOLO uses something called anchors, which is just a fancy way of saying that whatever the `xy` output, we are taking the sigmoid of the output (which normalizes the coordinates to within the grid) and then add the output onto the top left corner of the grid’s original image coordinate. The output width and height are also normalized by p_w and p_h, which are the width and height anchor dimensions of the box respectively. Below is a good visual depiction of this transform, which I plagiarized from the YOLOv3 paper, which plagiarized it from someone else.

![Transforms](/assets/yolo/transforms.png)

Now back to our implementation, what we want to do then is simply apply the transformations I previously mentioned onto the current layer, and output them as detections.

![YOLO transforms implementation](/assets/yolo/yolo_transform.png)

An odd thing I didn’t really understand from the config file is that there are two parameters defined, mask and anchors. Anchors is a long list of tuples, and the anchors we want to actually use are the ones corresponding to the indices specified by mask. 

![YOLO whack](/assets/yolo/yolo_parse.png)

Leaving that detail aside, we now get three detection outputs done on three different scales. I then concatenate these outputs as one tensor of shape [`number of predictions`, `B x (5 + C)`] as the final output.

![YOLO return](/assets/yolo/yolo_return.png)

I hope it’s clear how we are formulating the network so far, because we are done with the actual YOLO architecture itself!

## Loading weights

The way I tested whether my architecture had no major problems was to first feed it an image. Of course, the outputs would be meaningless since all the model weights are randomly initialized, but it was a nice check that the model worked smoothly and the returned shape is as we expected.

```
darknet = DarkNet()
output = darknet.build(img) # check the outputs
```

Now, to validate that our implementation actually works, it would be nice to set up an inference pipeline, and actually run the model on some test images. Thankfully, the author Joseph Redmon also included the weights they used in his [webpage](https://pjreddie.com/darknet/yolo/). So all we need to do is to load the weight file and check that our inferences look somewhat decent.

I did meet an issue at this point, though. The above is what I thought would happen in practice: I would build the model, then return the outputs in the format we specified above. However, I never considered how we might load the weights into the model in the first place — I just assumed that we can later set up a function like `model.load_weights()` and pass in the weights file. There were two issues I need to figure out: firstly, Tensorflow seems to only only support loading weights from either a ckpt file or a frozen graph. So my first issue was to find out how to convert the downloaded weight file into one of those two formats. Secondly, at least to my understanding, the way I use a Tensorflow model after loading the weight seems to be running a session, then getting a tensor whose name is defined in the ckpt/frozen graph file (which looked quite ugly to me — if I knew this earlier, I would probably design the YOLO class a bit differently. I might make some architecture revisions before posting the code on GitHub though).

### Converting weights to checkpoints

For the first problem, I kinda stole code (oops) from this tutorial [here](https://itnext.io/implementing-yolo-v3-in-tensorflow-tf-slim-c3c55ff59dbe) to convert the model weights file to the Tensorflow ckpt format. It seems that the author uses an older version of Tensorflow, though, but I solved that by importing `import tensorflow.compat.v1 as tf` and setting `tf.disable_v2_behavior()`. The tutorial also goes over the format of the provided weight file, but essentially all we needed to do was parse the weight file into loading ops, which we can then pass into a Tensorflow saver to convert into a bunch of checkpoint files. I probably won’t go over the details in this blog (since I was essentially just following along the tutorial), but do check it out [here](https://itnext.io/implementing-yolo-v3-in-tensorflow-tf-slim-c3c55ff59dbe)!

### Tensorflow sessions and tensors

For my second problem, rather than redesigning my YOLO class, I just defined a new tensor that is the identity of our current output — as you can probably tell, at this point I was getting kinda impatient to see some results, and started slacking off with my code design. It does work and I managed to get the output tensor by its name, but I like to think I paid the price of my dignity as a coder, and would definitely need to update my software structure (and this blog) in the future.

![Identity naming](/assets/yolo/identity.png)

Nevertheless, at this point, we have fully defined YOLOv3, loaded with the weights that the authors themselves used. Given an input image, we can get the model outputs like so:

![Load outputs](/assets/yolo/load_outputs.png)

Just a tip, if you're unsure of what your tensor name may be, you can always list all the names in your graph by checking
```
[n.name for n in tf.get_default_graph().as_graph_def().node]
```

## Post-processing

But we still need to parse the results into a comprehensible format. So we still need two steps: firstly, since I’m going to be using `cv2.rectangle` for drawing my bounding boxes, I wanted to convert YOLOv3’s `xywh` output format into a typical bounding box’s top left-bottom right coordinate format. That’s quite simple:

![Get bounding boxes](/assets/yolo/get_bb.png)

### Non-maximum suppression

And secondly, we want to apply non-maximum suppression to the output. Why is that so? Recall that how YOLO works is that each grid makes B bounding box predictions, for each anchor defined. This basically generates a ton of predictions, most of which overlap on the same object (assuming the number of objects in the image are not too numerous). So we simply use NMS to sort through the overlapping bounding boxes and find the tightest one. Truth be told, since NMS is quite straightforward and I was getting really impatient, I yet again stole the NMS function of [another YOLO implementation](https://github.com/mystic123/tensorflow-yolo-v3/blob/master/utils.py). 

And we’re done! Now we have the filtered bounding boxes for the predictions YOLOv3 made, along with the predicted classes and confidence on those objects. All that’s left is to draw this onto the original image (or for me, I simply overlay the predictions of the transformed image before transforming it back into its original size). The output is shown below:

![Get bounding boxes](/assets/yolo/test.png)

## Remarks

This was quite an exciting project for me, honestly, since I was more focused on the entire learning process rather than showing results, in comparison to schoolwork or internships. Unlike school and internships, I also didn’t have the human resource to refer to, such as upperclassmen, TAs, or mentors, which made it all the more rewarding. Moving forward, I definitely want to work on implementing other SOTA object detection models (particularly Mask-RCNN), but also some human-object pose estimation papers I’ve been quite interested in reading recently. I’ll probably talk more about that in a different post though!

### A note on training

As a side note, since this post was focused on implementing the YOLOv3 architecture, I slightly brushed over details on setting up the inference pipeline, and didn't actually implement the training pipeline or the loss function. For those who are curious, here's a screenshot of the loss function defined in the paper:

![Loss function](/assets/yolo/loss.png)

where `1_{i}` denotes if the object appears in cell `i`, and `1_{ij}` denotes if the `j`th bounding box predictor of cell `i` is responsible for that prediction. It is essentially the sum of the SSE for classification loss, localization loss (difference between predicted bounding box and ground truth box), and confidence loss.

One thing that has been bugging me is that if I were to implement more models in the future, it'd be great if I could come up with a framework for model training, evaluation, and inference, that would ideally be extensible across various kinds of models. That way, in future blog posts, I can just focus on defining model architectures, and have a standardized input and output abstraction ready for running trainings and inference. I don’t really have a solid idea of how that would look like yet, though, but hopefully I’ll have a better idea once I start implementing more models and understand where to draw that abstraction. Till then, thanks for reading!