---
layout: post
title: Compile Tensorflow C++ without Bazel
---

As the title, in this post I will give the instructions on how to compile Tensorflow C++ codes without Bazel. The reason why 
I wrrite this blog is because officially, to compile a C++ Tensorflow project, you have to integrate it in the source tree of 
tensorflow, create a BUILD file and compile it with bazel. An example is the label_image project under tensorflow/examples.

Sometime you want to create a C++ Tensorflow projects in your favorite C++ IDEs and build it Makefile of CMake, you will need
to do some extra work to allow gcc to compile successfully C++ Tensorflow codes. 

So this post aims to give detailed instructions on how to compile C++ Tensorflow codes with gcc. I will call it Standalone C++ 
Tensorflow. To test the latest version of tensorflow on the latest stable Ubuntu until now, I create a docker image with:
  -  **Ubuntu 17.10**
  -  **gcc 7.2**
  -  **tensorflow 1.4.0**
 
 The reason why I use Docker is to create an independent environment to test the latest version of tensorfow, because you can
 easily mess arround with all the includes and libs that already exist in your working environment. By creating a new Ubuntu
 docker image, it will be easier for you to follow the instructions without having to questions like: "Did you install library 
 XYZ?" or "What are your environment paths?". 
 
 I assume that you know the basic of Docker, here I create a ubuntu 17.10 image with this Dockerfile:
 ```
 FROM ubuntu:17.10

# Install.
RUN \
  sed -i 's/# \(.*multiverse$\)/\1/g' /etc/apt/sources.list && \
  apt-get update && \
  apt-get -y upgrade && \
  apt-get install -y build-essential && \
  apt-get install -y software-properties-common && \
  apt-get install -y byobu curl git htop man unzip vim wget && \
  rm -rf /var/lib/apt/lists/*
 ``` 
 Then use the following command to create the image:
 ```bash
 docker build -t ubuntu:17.10 .
 ```
 
**From now on, we will work on the interactive shell of docker image**, to go into the shell:
```bash
docker run -it ubuntu:17.10 /bin/sh
```
Install python
```
apt-get update && apt-get install python-dev python-pip python-setuptools python-sphinx python-yaml python-h5py python3-pip python-numpy python-scipy python-nose
```
You need to install cmake 3.9.6 to compile others libraries:
```
wget https://cmake.org/files/v3.9/cmake-3.9.6.tar.gz
tar -xzvf cmake-3.9.6.tar.gz
cd cmake-3.9.6
./configure
make
make install
```
Install googletest from sources
```
cd /home
git clone https://github.com/google/googletest.git
cd googletest
mkdir build
cd build
cmake ..
make
make install
```
Install protobuf from sources:
```
cd /home
apt-get install autoconf automake libtool
git clone https://github.com/google/protobuf.git
cd protobuf
./autogen.sh
./configure
make
make check
make install
ldconfig
```
Install Eigen 3.3.4 from sources
```
cd /home
wget http://bitbucket.org/eigen/eigen/get/3.3.4.tar.bz2
tar -xzvf 3.3.4.tar.bz2
cd eigen-folder
mkdir build
cd build
cmake ..
make
make install
```
Install bazel to compile tensorflow:
```
apt-get install openjdk-8-jdk
echo "deb [arch=amd64] http://storage.googleapis.com/bazel-apt stable jdk1.8" | tee /etc/apt/sources.list.d/bazel.list
curl https://bazel.build/bazel-release.pub.gpg | apt-key add -
```
Clone tensorflow repo:
```
git clone https://github.com/tensorflow/tensorflow.git
```
Compile tensorflow for c++ (default parameters, python3)
```
cd tensorflow
./configure
bazel build //tensorflow:libtensorflow_cc.so
```
Create a standalone folder to test compilation of C++ tensorflow with gcc
```
cd /home
mkdir standalone
```
To compile tensorflow with gcc, it has to get all the header files required to compile. Create an include folder and copy header files to that folder:
```
export TENSORFLOW_DIR=/home/tensorflow
cd /home/standalone
mkdir include
cp -r $TENSORFLOW_DIR/tensorflow include/third_party/
cp -r $TENSORFLOW_DIR/bazel-genfiles/tensorflow include/third_party/
cp -r /home/protobuf/src/google include/third_party/
cp -r $TENSORFLOW_DIR/third_party/eigen3 include/third_party/
cp -r /home/eigen-folder/. include/third_party/eigen3/
cp -r include/third_party/eigen3/Eigen include/third_party/
```
Clone google nsync:
```
cd /home/standalone/include
git clone https://github.com/google/nsync.git
```
We need also 2 libraries libtensorflow_cc.so and libtensorflow_framework.so. Copy those libraries to /usr/local/lib:
```
cd /home/standalone
mkdir lib
cp -r $TENSORFLOW_DIR/bazel-bin/tensorflow/libtensorflow_cc.so /usr/local/lib
cp -r $TENSORFLOW_DIR/bazel-bin/tensorflow/libtensorflow_framework.so /usr/local/lib
```
Refresh shared library cache:
```
ldconfig
```
Copy C++ example label_image under tensorflow source tree to standalone folder:
```
cd /home/standalone
cp $TENSORFLOW_DIR/tensorflow/examples/label_image/main.cc .
cp -r $TENSORFLOW_DIR/tensorflow/examples/label_image/data .
```
Get inception model
```
cd /home/standalone/data
wget https://storage.googleapis.com/download.tensorflow.org/models/inception_v3_2016_08_28_frozen.pb.tar.gz
tar -xzvf inception_v3_2016_08_28_frozen.pb.tar.gz
```
Create Makefile in standalone folder with content:
```
CC = g++
CFLAGS = -std=c++11 -g -Wall -D_DEBUG -Wshadow -Wno-sign-compare -w
INC = -I/usr/local/include/eigen3
INC +=  -I./include/third_party
INC += -I./include
INC += -I./include/nsync/public/
LDFLAGS =  -lprotobuf -pthread -lpthread
LDFLAGS += -ltensorflow_cc -ltensorflow_framework

all: main

main:
        $(CC) $(CFLAGS) -o main main.cc $(INC) $(LDFLAGS)
run:
        ./main --image=./data/grace_hopper.jpg --graph=./data/inception_v3_2016_08_28_frozen.pb --labels=./data/imagenet_slim_labels.txt
clean:
        rm -f main
```
Now do:
```
make
make run
```
Normally, it wil output:
```
./main --image=./data/grace_hopper.jpg --graph=./data/inception_v3_2016_08_28_frozen.pb --labels=./data/imagenet_slim_labels.txt
2017-11-19 13:10:23.989200: I tensorflow/core/platform/cpu_feature_guard.cc:137] 
2017-11-19 13:10:25.205406: I main.cc:250] military uniform (653): 0.834306
2017-11-19 13:10:25.205491: I main.cc:250] mortarboard (668): 0.0218692
2017-11-19 13:10:25.205520: I main.cc:250] academic gown (401): 0.0103579
2017-11-19 13:10:25.205544: I main.cc:250] pickelhaube (716): 0.00800814
2017-11-19 13:10:25.205563: I main.cc:250] bulletproof vest (466): 0.00535088
```
To sum up, by following instructions, you can create an evironment with:
  -  Ubuntu 17.10
  -  gcc 7.2.0
  -  tensorflow 1.4.0
  -  Python 2 or 3
  -  cmake 3.9.6
  -  Eigen 3.3.4
  -  Protobuf (master branch)
  -  Googletest (master branch)
  -  bazel 

Then all the headerfiles are copied in a folder names standalone that is ready to compile C++ tensorflow codes.

I hope that this blog helps. If you have any question, feel free to contact me to: [phan@tuanphuc.com](mailto:phan@tuanphuc.com).