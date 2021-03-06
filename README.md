# ROS2 TensorRT Wrapper

## Information

This repository is meant for inferencing neural networks in [ONNX](https://onnx.ai) format using the optimized [TensorRT](https://developer.nvidia.com/tensorrt) framework by NVIDIA, in as clean and as stand-alone format as possible, much like any other ROS package projects.

The general usage would be to build a TensorRT Engine from a pretrained ONNX model, using **onnx2tensorrt**, while subscribing to input, inferencing and publishing results would be handled by **ros2inference**.

**IMPORTANT**: TensorRT engines are optimized specifically to the device it is built on, therefore an engine built on your machine will not work on your robot running a Jetson. You will have to build to engine on the Jetson and use it.

### Benchmarks

In my past experience, performance boost from TensorRT optimization becomes increasingly obvious as the number of parameters and operations of the model goes up. Here are some comparisons between different aspects of inference performances on my personal machine which has a GTX1050TI, using the latest Pytorch, on the same pretrained ResNet-50 model.  (Code under tests)

Averaging on 1000 iterations after warmup on the GPU, ResNet-50,  

| Framework    | GPU Memory  | Inference Frequency |
| ------------ | ----------- | -------------------- |
| Pytorch      | 718 Mb      | 105.33 Hz            |
| **TensorRT** | **358 Mb**  | **197.55 Hz**        |

### Notes

- [ONNX](https://onnx.ai/) models can be obtained from pretrained Pytorch models (my personal preference) or Tensorflow etc, however depending on the complexity of the model, the current NVIDIA TensorRT framework might not accomodate all the layers, hence the need for additional plugins when building the network.
- [TensorRT](https://developer.nvidia.com/tensorrt) is a neural network optimization framework by NVIDIA, which optimizes runtime and space complexity of neural network inferencing on NVIDIA GPUs.
- [ROS2](https://index.ros.org/doc/ros2/) is used for its amazing real-time system potential, and allows for quality of service toggling, which will be useful in any real-time robotics systems, more features will be added bit by bit.
- [OpenCV](https://github.com/opencv/opencv) is used for encoding and decoding JPGs, which is currently only used certain parts of the tests, but will be used more in the future when full computer-vision-only-features are added. It is statically linked so it wont mess with your own machine's build.
- The project uses [std_msgs/Float32MultiArray](http://docs.ros.org/melodic/api/std_msgs/html/msg/Float32MultiArray.html) message types instead of [sensor_msgs/CompressedImage](http://docs.ros.org/melodic/api/sensor_msgs/html/msg/CompressedImage.html) to allow generality, so it can be used in audio or text applications as well, with minor complexity drawbacks.
- Take note on how the float arrays are created, especially when using images, for example JPGs decoded by OpenCV will be BGR instead of RGB, while there may be additional normalization techniques for inferencing, such an example is shown in the image publishing testing projects **src/tests/test_publisher.hpp**.
- Make sure to properly resize, crop or normalize your data before passing to this project for inference, otherwise your results might be undefined, check out the example below and the code on how I normalized the data and shuffled the channels appropriately based on Imagenet training.
- Please note that this project will not work on **ARM** architecture devices (NVIDIA Jetsons, Xaviers), but should work seamlessly with **x86** architectures (Desktops, Laptops, MiniPCs), due to the need for cross-compilation (future work).

## Tested and rolling with:

- C++ 17 [Verified]
- Cmake 3.17.2 [Verified]
- gcc 7.5.0 [Verified]
- TensorRT 7.0.0.11 [Verified]
- CUDA 10.2 [Verified]
- NVIDIA Quadro M1200 [Verified]
- ROS2 Eloquent Elusor [Verified]

## Third-party Prerequisites (installation difficulty):

- CUDA (medium, remember to add path to system **PATH** and **LD_LIBRARY_PATH**!)
- ROS2 (medium)
- TensorRT (easy)

Add a path for TensorRT at the bottom of `$HOME/.bashrc`, for example
```
export TENSORRT_DIR=$HOME/TensorRT-7.0.0.11
```
this path will be used in building and linking the project.

## Building

```
git clone --recursive https://github.com/aaronchongth/ros2_tensorrt.git
./install_reqs
./build_deps
./build
```

## Usage

```
./bin/ros2inference --node-name [node_name_here] --sub-topic [array_topic_here] --pub-topic [output_topic_here] --model-path [ONNX_or_engine_model_path_here]
```

## Example
This example will validate whether the installations and inferences are correct, in 3 separate terminals, it publishes the CompressedImage and Float32MultiArray message type of a cat image, found in **data/cat_224.jpg**, inferences it through an Imagenet Pretrained Resnet50v1 model, which should have a prediction of a Tabby cat, index 281.

Download the Imagenet pretrained ONNX model, build the TensorRT engine,
```
wget https://s3.amazonaws.com/onnx-model-zoo/resnet/resnet50v1/resnet50v1.onnx -P data
./bin/onnx2tensorrt --model-path data/resnet50v1.onnx --output-path data/resnet50v1.engine
```

Terminal 1 - Handles the cat image, normalizes it, shuffles the channels properly and publishes Float32MultiArray as well as CompressedImage.
```
./bin/test_publisher --array-topic array_topic
```

Terminal 2 - Inferences array data and publishes raw output from neural network.
```
./bin/ros2inference --sub-topic array_topic --pub-topic output_topic --model-path data/resnet50v1.engine
```

Terminal 3 - Subscribes to raw output and performs an argmax to get the prediction index.
```
./bin/test_subscriber --sub-topic output_topic
```

## Cleaning, rebuilding

```
./clean
./build
```

## Cleaning all, including built dependencies, rebuilding

```
./clean_deps
./build_deps
./build
```

## To do:
- bash script for tests
- use json executables
- handle script for downloading example onnx model
- try using libjpegturbo to encode and decode instead of OpenCV
- add post processing plugin
- cross compilation flag for ARM architectures!
- ROS version of this!

## Credits
- Models from examples and tests used are from [pretrained models](https://github.com/onnx/models/tree/master/models/image_classification/resnet)
- [ONNX](https://onnx.ai)
- [TensorRT](https://developer.nvidia.com/tensorrt)
- [OpenCV](https://github.com/opencv/opencv)
- [ROS2](https://index.ros.org/doc/ros2/)

## License
MIT
