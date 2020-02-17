# pytorch-blender

Seamless integration of Blender renderings into [PyTorch](http://pytorch.org) datasets for deep learning from artificial visual data. This repository contains a minimal demonstration that harvests images and meta data from ever changing Blender renderings.

```
python pytorch_sample.py
```
renders a set of images of random rotated cubes to `./tmp/output_##.png`, such as the following

![](etc/result.png)

This image is generated by 4 Blender instances, randomly perturbating a minimal scene. The results are collated in a `BlenderDataset`. A PyTorch `DataLoader` with batch size 4 is used to grab from the dataset and produce the figure.

The code accompanies our [academic work](https://arxiv.org/abs/1907.01879) in the field of machine learning from artificial images. When using please cite the following work
```
@misc{cheindkpts2019,
Author = {Christoph Heindl and Sebastian Zambal and Josef Scharinger},
Title = {Learning to Predict Robot Keypoints Using Artificially Generated Images},
Year = {2019},
Eprint = {arXiv:1907.01879},
Note = {To be published at ETFA 2019},
}
```

## Code outline
```Python
import torch.utils.data as data

from blendtorch.torch import BlenderLauncher, Receiver

# Standard PyTorch Dataset convention
class MyDataset:

    def __init__(self, blender_launcher, transforms=None):
        self.recv = Receiver(blender_launcher)
        self.transforms = transforms

    def __len__(self):
        # Virtually anything you'd like to end episodes.
        return 100 

    def __getitem__(self, idx):        
        # Data is a dictionary of {image, xy, id}, 
        # see publisher script
        d = self.recv(timeoutms=5000)   
        return d['image'], d['xy'], d['btid']

kwargs = {
    'num_instances': 4,         # Number of parallel instances
    'scene': 'scene.blend',     # Scene to render in Blender
    'script': 'blender.py',     # Script to run in Blender
    'blend_path': '.',          # Additional path to look for Blender
}

with BlenderLauncher(**kwargs) as bl:        
    ds = MyDataset(bl)        
    dl = data.DataLoader(ds, batch_size=4, num_workers=0)

    gen = iter(dl)

    for _ in range(10):
        (x, coords, ids) = next(gen)
        print(f'Received from {ids.numpy()}')        
```

## Runtimes

The runtimes for the demo scene (really quick to render) is shown below.

| Blender Instances  | Runtime ms/batch  |
|:-:|:-:|
| 1  | `103 ms ± 5.17 ms`  |
| 2  | `43.7 ms ± 10.3 ms` |

The above timings include rendering, transfer and encoding/decoding. Depending on the complexity of renderings you might want to tune the number of instances.

## Prerequisites
The following packages need to be available in your PyTorch environment and Blender environment:
 - Python >= 3.7
 - [Blender](https://www.blender.org/) >= 2.79
 - [PyTorch](http://pytorch.org) >= 0.4
 - [PyZMQ](https://pyzmq.readthedocs.io/en/latest/)
 - [Pillow/PIL](https://pillow.readthedocs.io/en/stable/installation.html)

Both packages are installable via `pip`. In order add packages to your Blender packaged Python distribution, execute the following commands (usually administrator privileges are required on Windows)

```
"<BLENDERPATH>2.79\python\bin\python.exe" -m ensurepip
"<BLENDERPATH>2.79\python\bin\python.exe" -m pip install pyzmq
"<BLENDERPATH>2.79\python\bin\python.exe" -m pip install pillow
```
where `<BLENDERPATH>` is the file path to the directory containing the Blender executable.

**Note** If `<BLENDERPATH>` is not in your `PATH` then set `blend_path=...` in `BlenderLauncher`.

## How it works
An instance of [BlenderLaunch](blendtorch/launcher.py) is responsible for starting and stopping background Blender instances. The script `blender.py` and additional arguments are passed to the starting Blender instance. `blender.py` creates a publisher socket for communication and starts producing random renderings. Meanwhile, a PyTorch dataset uses a [Receiver](blendtorch/receiver.py) instance to read data from publishers.

## Caveats
- In background mode, Blender `ViewerNodes` are not updated, so rendering have to be written to files. Currently, the Blender script re-imports the written image and sends it as message to any subscriber. This way, we do not need to keep track of which files have already been read and can be deleted, which simplifies communication.
- In this sample. only the main composite rendering is transmitted. You might want to use `FileOutputNode` instead, to save multiple images per frame.
- Currently you need to use `num_workers=0` when creating a PyTorch `DataLoader` as the `Receiver` object is capable of multi-process pickling.

