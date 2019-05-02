# Pywick

#### High-Level Training framework for Pytorch

This framework is based in large part on the excellent [Torchsample](https://github.com/ncullen93/torchsample) framework originally published by @ncullen93.


PyWick aims to provide a *batteries included* framework for training neural networks. Among other things it includes:
- State of the art normalization, activation, loss functions and optimizers not available in the standard Pytorch library.
- A high-level module for training with callbacks, constraints, metrics, conditions and regularizers.
- Dozens of popular object classification and semantic segmentation models.
- Comprehensive data loading, augmentation, transforms, and sampling capability.
- Utility tensor functions
- Useful meters
- Basic GridSearch (exhaustive and random)

## Install
Substitute version number as necessary:

`pip install git+https://github.com/achaiah/pywick.git@v0.5.2`

## ModuleTrainer
The `ModuleTrainer` class provides a high-level training interface which abstracts
away the training loop while providing callbacks, constraints, initializers, regularizers,
and more.

Example:
```python
from pywick.modules import ModuleTrainer
from pywick.initializers import XavierUniform
from pywick.metrics import CategoricalAccuracySingleInput
import torch.nn as nn
import torch.functional as F

# Define your model EXACTLY as normal
class Network(nn.Module):
    def __init__(self):
        super(Network, self).__init__()
        self.conv1 = nn.Conv2d(1, 32, kernel_size=3)
        self.conv2 = nn.Conv2d(32, 64, kernel_size=3)
        self.fc1 = nn.Linear(1600, 128)
        self.fc2 = nn.Linear(128, 10)

    def forward(self, x):
        x = F.relu(F.max_pool2d(self.conv1(x), 2))
        x = F.relu(F.max_pool2d(self.conv2(x), 2))
        x = x.view(-1, 1600)
        x = F.relu(self.fc1(x))
        x = F.dropout(x, training=self.training)
        x = self.fc2(x)
        return F.log_softmax(x)

model = Network()
trainer = ModuleTrainer(model)   # optionally supply cuda_devices as a parameter

initializers = [XavierUniform(bias=False, module_filter='fc*')]

# initialize metrics with top1 and top5 
metrics = [CategoricalAccuracySingleInput(top_k=1), CategoricalAccuracySingleInput(top_k=5)]

trainer.compile(loss='cross_entropy',
                # callbacks=callbacks,          # define your callbacks here (e.g. model saver, LR scheduler)
                # regularizers=regularizers,    # define regularizers
                # constraints=constraints,      # define constraints
                optimizer='sgd',
                initializers=initializers,
                metrics=metrics)

trainer.fit_loader(train_dataset_loader, 
            val_loader=val_dataset_loader,
            num_epoch=20,
            verbose=1)
```
You also have access to the standard evaluation and prediction functions:

```python
loss = trainer.evaluate(x_train, y_train)
y_pred = trainer.predict(x_train)
```
PyWick provides a wide range of <b>callbacks</b>, generally mimicking the interface
found in `Keras`:

- `CSVLogger` - Logs epoch-level metrics to a CSV file
- [`CyclicLRScheduler`](https://github.com/bckenstler/CLR) - Cycles through min-max learning rate
- `EarlyStopping` - Provides ability to stop training early based on supplied criteria
- `History` - Keeps history of metrics etc. during the learning process
- `LambdaCallback` - Allows you to implement your own callbacks on the fly
- `LRScheduler` - Simple learning rate scheduler based on function or supplied schedule
- `ModelCheckpoint` - Comprehensive model saver
- `ReduceLROnPlateau` - Reduces learning rate (LR) when a plateau has been reached
- `SimpleModelCheckpoint` - Simple model saver
- Additionally, a `TensorboardLogger` is incredibly easy to implement via the [TensorboardX](https://github.com/lanpa/tensorboardX) (now part of pytorch 1.1 release!)
library but is not included here to reduce the number of dependencies


```python
from pywick.callbacks import EarlyStopping

callbacks = [EarlyStopping(monitor='val_loss', patience=5)]
trainer.set_callbacks(callbacks)
```

PyWick also provides <b>regularizers</b>:

- `L1Regularizer`
- `L2Regularizer`
- `L1L2Regularizer`


and <b>constraints</b>:
- `UnitNorm`
- `MaxNorm`
- `NonNeg`

Both regularizers and constraints can be selectively applied on layers using regular expressions and the `module_filter`
argument. Constraints can be explicit (hard) constraints applied at an arbitrary batch or
epoch frequency, or they can be implicit (soft) constraints similar to regularizers
where the the constraint deviation is added as a penalty to the total model loss.

```python
from pywick.constraints import MaxNorm, NonNeg
from pywick.regularizers import L1Regularizer

# hard constraint applied every 5 batches
hard_constraint = MaxNorm(value=2., frequency=5, unit='batch', module_filter='*fc*')
# implicit constraint added as a penalty term to model loss
soft_constraint = NonNeg(lagrangian=True, scale=1e-3, module_filter='*fc*')
constraints = [hard_constraint, soft_constraint]
trainer.set_constraints(constraints)

regularizers = [L1Regularizer(scale=1e-4, module_filter='*conv*')]
trainer.set_regularizers(regularizers)
```

You can also fit directly on a `torch.utils.data.DataLoader` and can have
a validation set as well :

```python
from pywick import TensorDataset
from torch.utils.data import DataLoader

train_dataset = TensorDataset(x_train, y_train)
train_loader = DataLoader(train_dataset, batch_size=32)

val_dataset = TensorDataset(x_val, y_val)
val_loader = DataLoader(val_dataset, batch_size=32)

trainer.fit_loader(loader, val_loader=val_loader, num_epoch=100)
```

## Data Augmentation and Datasets
The PyWick package provides a ton of good data augmentation and transformation
tools which can be applied during data loading. The package also provides the flexible
`TensorDataset`, `FolderDataset` and 'MultiFolderDataset' classes to handle most dataset needs.

### Torch Transforms
##### These transforms work directly on torch tensors

- `AddChannel`
- `ChannelsFirst`
- `ChannelsLast`
- `Compose`
- `ExpandAxis`
- `Pad`
- `PadNumpy`
- `RandomChoiceCompose`
- `RandomCrop`
- `RandomFlip`
- `RandomOrder`
- `RangeNormalize`
- `Slice2D`
- `SpecialCrop`
- `StdNormalize`
- `ToFile`
- `ToNumpyType`
- `ToTensor`
- `Transpose`
- `TypeCast`

##### Additionally, we provide image-specific manipulations directly on tensors:

- `Brightness`
- `Contrast`
- `Gamma`
- `Grayscale`
- `RandomBrightness`
- `RandomChoiceBrightness`
- `RandomChoiceContrast`
- `RandomChoiceGamma`
- `RandomChoiceSaturation`
- `RandomContrast`
- `RandomGamma`
- `RandomGrayscale`
- `RandomSaturation`
- `Saturation`

#####  Affine Transforms (perform affine or affine-like transforms on torch tensors)

- `RandomAffine`
- `RandomChoiceRotate`
- `RandomChoiceShear`
- `RandomChoiceTranslate`
- `RandomChoiceZoom`
- `RandomRotate`
- `RandomShear`
- `RandomSquareZoom`
- `RandomTranslate`
- `RandomZoom`
- `Rotate`
- `Shear`
- `Translate`
- `Zoom`

We also provide a class for stringing multiple affine transformations together so that only one interpolation takes place:

- `Affine` 
- `AffineCompose`

##### Blur and Scramble transforms (for tensors)
- `Blur`
- `RandomChoiceBlur`
- `RandomChoiceScramble`
- `Scramble`

### Datasets and Sampling
We provide the following datasets which provide general structure and iterators for sampling from and using transforms on in-memory or out-of-memory data. In particular,
the [FolderDataset](pywick/datasets/FolderDataset.py) has been designed to fit most of your dataset needs. It has extensive options for data filtering and manipulation.
It supports loading images for classification, segmentation and even arbitrary source/target mapping. Take a good look at its documentation for more info.

- `ClonedDataset`
- `CSVDataset`
- `FolderDataset`
- `MultiFolderDataset`
- `TensorDataset`
- `tnt.BatchDataset`
- `tnt.ConcatDataset`
- `tnt.ListDataset`
- `tnt.MultiPartitionDataset`
- `tnt.ResampleDataset`
- `tnt.ShuffleDataset`
- `tnt.TensorDataset`
- `tnt.TransformDataset`

### Imbalanced Datasets
In many scenarios it is important to ensure that your traing set is properly balanced,
however, it may not be practical in real life to obtain such a perfect dataset. In these cases 
you can use the `ImbalancedDatasetSampler` as a drop-in replacement for the basic sampler provided
by the DataLoader. More information can be found [here](https://github.com/ufoym/imbalanced-dataset-sampler)

```python
from pywick.samplers import ImbalancedDatasetSampler

train_loader = torch.utils.data.DataLoader(train_dataset, 
    sampler=ImbalancedDatasetSampler(train_dataset),
    batch_size=args.batch_size, **kwargs)
```

## Extensive Library of Image Classification Models (most are pretrained!)
- **All standard models from Pytorch (Densenet, ResNet, VGG)**
- **BatchNorm Inception**
- **Dual-Path Networks**
- **FBResnet**
- **Inception v3**
- **Inception v4**
- **InceptionResnet v2**
- **MobileNet v2**
- **NasNet and NasNet Mobile** ([Learning Transferable Architectures for Scalable Image Recognition](https://arxiv.org/abs/1707.07012))
- **PNASNet**
- **Polynet**
- **Pyramid Resnet**
- **Resnet**
- **Resnet + Swish**
- **ResNext**
- **SE Net**
- **SE Inception**
- **ShuffleNet v2**
- **Wide Resnet**
- **XCeption**

## Image Segmentation Models
1. **Vanilla FCN:** FCN32, FCN16, FCN8, in the versions of VGG, ResNet and DenseNet respectively
([Fully convolutional networks for semantic segmentation](http://www.cv-foundation.org/openaccess/content_cvpr_2015/papers/Long_Fully_Convolutional_Networks_2015_CVPR_paper.pdf))
2. **U-Net** ([U-net: Convolutional networks for biomedical image segmentation](https://arxiv.org/pdf/1505.04597))
3. **SegNet** ([Segnet: A deep convolutional encoder-decoder architecture for image segmentation](https://arxiv.org/pdf/1511.00561))
4. **PSPNet** ([Pyramid scene parsing network](https://arxiv.org/pdf/1612.01105))
5. **GCN** ([Large Kernel Matters](https://arxiv.org/pdf/1703.02719))
6. **DUC, HDC** ([understanding convolution for semantic segmentation](https://arxiv.org/pdf/1702.08502.pdf))
7. **Tiramisu** ([The One Hundred Layers Tiramisu: Fully Convolutional DenseNets for Semantic Segmentation](https://arxiv.org/pdf/1611.09326))
8. **Deeplab v2** ([DeepLab: Semantic Image Segmentation with Deep Convolutional Nets, Atrous Convolution, and Fully Connected CRFs](https://arxiv.org/abs/1606.00915))
9. **Deeplab v3** ([Rethinking Atrous Convolution for Semantic Image Segmentation](https://arxiv.org/abs/1706.05587))
10. **RefineNet** ([RefineNet](https://arxiv.org/abs/1611.06612))
11. **FusionNet** ([FusionNet in Tensorflow by Hyungjoo Andrew Cho](https://github.com/NySunShine/fusion-net))
12. **ENet** ([ENet: A Deep Neural Network Architecture for Real-Time Semantic Segmentation](https://arxiv.org/abs/1606.02147))
13. **LinkNet** ([Link-Net](https://codeac29.github.io/projects/linknet/))
14. **FRRN** ([Full Resolution Residual Networks for Semantic Segmentation in Street Scenes](https://arxiv.org/abs/1611.08323))
15. Additional variations of many of the above

###### To load one of these models:
```python
# use the `get_model` utility
from pywick.models.model_utils import get_model, ModelType

model = get_model(model_type=ModelType.CLASSIFICATION, model_name='resnet18', num_classes=1000, pretrained=True)
```
For a complete list of models you may want to take a look at the respective `pywick.models.[`[classification](pywick/models/classification/__init__.py) / [segmentation](pywick/models/segmentation/__init__.py)`].__init__` file


## Utility Functions
PyWick provides a few utility functions not commonly found:

### Tensor Functions
- `th_iterproduct` (mimics itertools.product)
- `th_gather_nd` (N-dimensional version of torch.gather)
- `th_random_choice` (mimics np.random.choice)
- `th_pearsonr` (mimics scipy.stats.pearsonr)
- `th_corrcoef` (mimics np.corrcoef)
- `th_affine2d` and `th_affine3d` (affine transforms on torch.Tensors)


## Acknowledgements and References
##### Thank you to the following people and the projects they maintain:
- @ncullen93
- @cadene
- @deallynomore
- @recastrodiaz
- @zijundeng
- And many others! (attributions listed in the codebase as they occur)

##### Thank you to the following projects from which we gently borrowed code and models
- [PyTorchNet](https://github.com/pytorch/tnt)
- [pretrained-models.pytorch](https://github.com/Cadene/pretrained-models.pytorch)
- [DeepLab_pytorch](https://github.com/doiken23/DeepLab_pytorch)
- [Pytorch for Semantic Segmentation](https://github.com/zijundeng/pytorch-semantic-segmentation)
- [Binseg Pytorch](https://github.com/saeedizadi/binseg_pytoch)
- And many others! (attributions listed in the codebase as they occur)



| *Thangs are broken matey! Arrr!!!* |
|-----------------------|
| I'm working on this project as time permits so you might discover bugs here and there. Feel free to report them, or better yet, to submit a pull request! |