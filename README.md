This repository contains code used in the 2020 SIIM-ISIC Melanoma Detection Competition on Kaggle. The code implements an idea of mine that could expanded upon to yield good results on other vision tasks, therefore I decided I should document my findings. Although I did not create a model that is highly competitive in terms of the competition, I did manage to come up with a method that can achieve promising performance while using small image sizes, and small image models. This allowed me to run experiments on my personal hardware without requiring a prohibitive training time. The code in this repository is in a working state and was only used for this specific experiments I ran for the competition. Therefore, it may not be easy to reproduce my results directly, but the repo should still serve to disseminate the general idea.

The task at hand is that of melanoma classification given an image of a skin lesion. Due to the simplicity of this task, I was able to rapidly prototype ideas while also tracking my progress via the online leaderboard. 

### Disclaimer
After working on this project I searched around a bit for related work, and found:

[Diagnose like a Radiologist: Attention Guided Convolutional Neural Network for Thorax Disease Classification](https://arxiv.org/pdf/1801.09927.pdf)

This paper uses a eerily similar approach, although it doesn't seem like they use the cropping technique to cross-reference original high-resolution images, and their attention technique varies from mine. Credit should definitely go to these authors for first experimenting with this idea. I also assume many others have tried methods like this (if so please let me know so I can add credit). I figured since I put time into this approach I would still document my findings.

### Usage
A lot of code is included as part of the repo, much of it is not very useful without the actual data (although all the preprocessing code is provided). I have included a small notebook displaying the attention guided cropping on some example test data using pre-trained weights. Here are brief descriptions of the files included:
```
│   crop.py -- code to apply attention cropping to specified dataset
│   dataset.py -- contains custom dataset from laoding images
│   preprocess.py -- contains code for preprocessing dataset for experiments
│   README.md -- this file
|   requirements.txt -- requirements generated by my conda environment
│   resnet.py -- contains all of the model code
│   train.py -- contains main training loop for all experiments
│   util.py -- miscellaneous functions (loss, augmentations, etc.)
│
├───notebooks
    │   attn_visualization.ipynb -- notebook displaying attention guided cropping
    │
    └───example_data
            chipnet_weights.pt -- pretrained weights for the chipping model
            test.csv -- metadata file for the test data
            test_subset.npy -- the first 300 image of the test set 

```

The most useful of these files is `attn_visualization.ipynb`, simply start up a jupyter notebook and run the cells to see a visual example of the attention guided cropping technique. For the other files, the arguments can be found by running the file with a `--help` flag. Each file is commented pretty well which should also help to explain any details left out of this README file.

### Approach

The dermatoscopy images provided as part of the competition are very high resolution, the largest images in the dataset are 4000x6000 pixels. When using pre-trained networks, the images for the downstream task are typically resized to match the size of the pre-training task. For example when using an ImageNet pre-trained CNN, one will typically resize images to 224 x 224, as this is what the network is "used" to. As many found out during the competition, this was not necessary due to scale invariance, as well as the vast difference between pre-training tasks and the competition task. Participants began ensembling models trained on various image sizes, anywhere from 224 x 224 all the way up to 1024 x 1024. Due to my limited resources, and aversion to online notebooks, I decided I wouldn't really be able to compete with large models and large image sizes. I instead made a personal challenge to squeeze as much performance as I could using only 224 x 224 images. 

When examining the images, I observed that in many images, the lesion in question occupies a very small portion of the image. This means that a large part of the 224^2 pixels (presumably) do not provide a significant amount of information. I realized I could gain back information by cropping the images tightly around the lesion and before resizing, this would result in a much higher resolution version of the lesion while maintaining the 224 x 224 image size! 

![example of targeted image crop to increase lesion detail](https://github.com/CCareaga/attention_guided_cropping/blob/master/figures/cropping.png?raw=true)

The amount of space occupied by the lesion varies greatly from image to image and other noise exists in the images (shadows, rulers, etc.) so this process cannot be trivially automated despite the simplicity of the images. I decided the best way to find salient regions is by having a model decide what information is important for classification. To accomplish this goal I decided to try to use attention. In this case I figured it would be pretty easy for the model to decide which pixels are lesion pixels and which pixels are regular skin without requiring explicit supervision. I devised the following model as a first attempt:

![initial attention model](https://github.com/CCareaga/attention_guided_cropping/blob/master/figures/model_diagram.png?raw=true)

The model is a simple extension of the ResNet architecture, allowing for the use of pre-trained weights. The changes consist of removing the global average pooling from the end of the ResNet and replacing it with an attention-weighted pooling operation. The attention weights are determined by a series of two convolutional layers over the unpooled feature maps to produce a single 7x7 map of scores. The scores are softmax for two reasons:

1. They cause attention weights to sum to 1 allowing for a simple weighted average.
2. They force the model to "choose" which pixels it wants to keep while attenuating the rest.

The second point is what makes the model "focus" on certain regions. The model can still predict a uniform attention map, but when it does, the average pooling "mixes" the information causing a noisier representation. When the model predicts a concentrated attention map, the pooling selects the pixel features coming from discriminative regions of the image (in this case the skin lesion). All of this attention learning is driven by supervision from the task at hand, meaning that the regions that are selected are deemed important to the classification of the lesion. This means we can now extract salient regions from the image, as well as diagnose any spurious correlations picked up by the model.

This initial attempt worked surprisingly well, the model did in fact determine important regions of the image, but unfortunately the attention maps where not exact. I conjectured that this was because spatial locations of the input don't perfectly correspond to the same spatial locations of the resulting feature maps. This is due to the large effective receptive field size of each feature map location. To fix this issue, I devised a way to constrict the information that contributes to each location of feature map. I decided to divide the image into square pieces, referred to as chips,  and feed each one into a ResNet separately. With the global average pooling, this results in a 512 dimensional embedding for each chip. I could then concatenate the chips back together to create a feature map where each position corresponding directly to a fixed region of the input image:

![improved attention model](https://github.com/CCareaga/attention_guided_cropping/blob/master/figures/chip_diagram.png?raw=true)

This change resulted in much more consistent and accurate attention maps such as the following:
![examples of good attention maps generated](https://github.com/CCareaga/attention_guided_cropping/blob/master/figures/attn_examples.png?raw=true)

These are some selected samples of the attention maps and their corresponding input images. It is obvious that the model is focusing on the pixels that contain the legion. While the maps look satisfactory a majority of the time, there are failure cases:
![examples of bad attention maps generated](https://github.com/CCareaga/attention_guided_cropping/blob/master/figures/failures.png?raw=true)

I was not able to track down why these images generate poor attention maps, but I did observe that larger models greatly improved the consistency of the attention maps. The examples shown above were generated using a ResNet-50 backbone and the chipping model discussed earlier. Once I was able to generate satisfactory attention maps, I developed a process to convert these attention maps into square regions for cropping:
![converting attention map to bounding box for cropping](https://github.com/CCareaga/attention_guided_cropping/blob/master/figures/attn_to_crop.png?raw=true)

When the attention fails the model generally generates a pretty unifrom attention map. This means that the bounding box created encapsulates a very large portion of the image and therefore the cropping process defaults to maintaining the original image. With cropped versions of both the training set and the test set, a new model can be trained. By ensembling the predictions of the a ResNet trained on the original images, and a ResNet trained on the cropped images, we can get a decent boost in performance without having to increase the image size!

### Implementation

#### Preprocessing

To start I center-cropped all images into a square using the size of their shortest side. Next, I resized each image to be 224 x 224 and store the training set and test set as .npy files (for easy loading into RAM). Before begin fed into any of the models, the images go through a series of augmentations:
```
train_transform = transforms.Compose([
    DrawHair(),
    transforms.RandomHorizontalFlip(),
    transforms.RandomVerticalFlip(),
    transforms.RandomRotation(360),
    transforms.ColorJitter(
        brightness=[0.8, 1.2],
        contrast=[0.8, 1.2],
        saturation=[0.8, 1.2],
    ),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406],std=[0.229, 0.224, 0.225])
])
```
All of these are built into PyTorch except the `DrawHair` transform, which is a simple class I wrote to draw random curved lines over the image simulating hair. 

The meta-data fields are converted to one-hot encodings. The one-hot encodings include an encoding for unknown/null. This process results in a 29-dimensional vector.

#### Models
Three models are trained as part of the previously described approach:
1. resnet: A regular ResNet trained on the original images and provided metadata
2. chipnet: The chip model used to generate attention maps
3. cropnet: A regular ResNet trained on images cropped using attention

Each of these models has a very similar training process. The training data consists of 33,126  images of varying size (32,542 benign, 584 malignant).  Metadata is included for each image, the meta-data fields are: age, anatomical site, gender, and specific diagnosis. These meta-data fields are converted to features and used as additional input to the models. Each model is trained and validated using 3-fold stratified cross-validation, this maintains the class distribution in each fold. The test set predictions of each fold are ensembled to produce the final predictions for each of the models. All models use focal loss to train, but would likely perform just as well with vanilla binary cross entropy. Additionally, test set predictions are generated by combining three rounds of test time augmentation using the augmentations described previously.

##### ResNet50 model (resnet)
This model consists of a ResNet50 to process the image input, along with a two linear layers to process the meta-data features. The embeddings from these two branches are concatenated and sent through a final linear layer to produce a single score. This model is implemented in `resnet.py` by the class `MetaResNet`. The only variation between this model and cropnet model is the images used to train. The model is trained using the Adam optimizer, and a balanced class sampler (oversampling), meaning each batch contains an even number of each class. The ResNet portion of the model is trained with a learning rate of 1x10 <sup>-5</sup> and the rest of the parameters have a learning rate of 5x10<sup>-5</sup>. Training proceeds until the validation AUC does not improve for 3 epochs. 

##### Chipping model (chipnet)
This model also consists of a ResNet50 backbone, but has no additional meta-data branch as it's goal is simply to produce attention maps. As explained before, each 224 x 224 image is split into 32 x 32 non-overlapping chips. Each chip is sent through the ResNet50 backbone and the resulting embeddings are re-stacked to create 7 x 7 feature maps with some number of channels depending on the backbone architecture. To compute attention maps, two 3 x 3 same convolutions are applied (with ReLU and batch norm in between). The result of this is a 7 x 7 map of scores. These scores are softmaxed to sum to one and used to take a weighted average of the original feature maps. This process results in a single embedding that can then be used for classification after the application of a final linear layer. This model is implemented in `resnet.py` by the class `ChipNet`. The model is trained in the same way as the resnet model except the ResNet backbone is trained with a learning rate of 5x10<sup>-5</sup> while the attention sub-net and final classifier are trained with a learning rate of 1x10<sup>-4</sup>.

##### Cropped image model (cropnet)
As previously stated, this model is trained identically to the resnet model except it uses the images cropped by the attention maps generated from the chipnet model.

### Results

I tried multiple ensembling techniques against the public leaderboard of the competition. Here are the results for the predictions of each model in the previously described approach:
|model|public LB score (AUC ROC)  |
|--|--|
|ResNet-50 on original images (resnet)| **0.9203** |
|Chip model (chipnet) | 0.8931|
|ResNet-50 on cropped images (cropnet) | 0.9177 |

The resnet model achieved the highest performance on the public leaderboard, while the chipnet model scored the lowest. This makes sense as the chipnet only uses disjoint tiles and therefore has a very constrained receptive field corresponding to each feature map location. This hinders performance while also improving attention map consistency and accuracy. The cropnet model achieves performance comparable to the resnet model, despite the attention maps sometimes producing poor crops of the original image. With a better chipnet model, I believe there would be a corresponding improvement in the performance of the cropnet model. I used these numbers to  generate ensemble weights for the each model's predictions (bad practice; leads to overfitting public LB), to see how much new information the chipnet and cropnet models add.

|ensemble|public LB|
|--|--|
|(0.4 * resnet) + (0.2 * chipnet) + (0.4 * cropnet) | 0.9263|
|(0.7 * resnet) + (0.3 * chipnet) | 0.9203 |
|(0.5 * resnet) + (0.5 * cropnet) | **0.9287** |

Turns out using the predictions of the chipnet does not improve results and the best ensemble is a straight average of the resnet model and the cropnet model. Although public LB may not be the best performance indicator, I argue that it at least shows that this method is viable and extra information can be gleaned from the cropped images. 

### Conclusion
Although the approach to medical image classification isn't necessarily novel, I believe there is still a lot of exploration to be done in this area. I could see these methods being applied to other vision tasks (for example landmark recognition). Additionally, I hope this repository is at least slightly informational and encourages others to experiment beyond the typical image classification approaches.
