# ppc64le Builds

Dockerfiles and wheel build scripts for building TF on ppc64le.

Maintainer: @wdirons (IBM)

* * *

`cpu/Dockerfile.manylinux_2014` extends from the
quay.io/pypa/manylinux2014_ppc64le docker image and installs java, bazel (from
source), and the minimal required pip packages to build TensorFlow. It also adds
the jenkins userid necessary to work with Oregon State University's Open Source
Lab jenkins environment. `gpu/Dockerfile.manylinux_2014.cuda10_1` extends from
the cpu image and adds CUDA 10.1 and cuDNN 7.6.

The bash scripts are the equivalent of what is run in the TensorFlow builds done
with Jenkins (everything except the remote cache server). They can also be run
manually by cloning tensorflow and copying the script into the root of the
workspace.

# Build on grid5000
@anji993

## Deploy node on drac
```
oarsub -I -l nodes=1,walltime=12 -q testing -t deploy -t destructive -t exotic
kadeploy3 -e ubuntu1804-ppc64-min -m drac-x.grenoble.grid5000.fr -p 5 -k
ssh root@drac-x.grenoble.grid5000.fr
```
## Install cuda
```
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1804/ppc64el/cuda-ubuntu1804.pin
sudo mv cuda-ubuntu1804.pin /etc/apt/preferences.d/cuda-repository-pin-600
wget https://developer.download.nvidia.com/compute/cuda/11.1.0/local_installers/cuda-repo-ubuntu1804-11-1-local_11.1.0-455.23.05-1_ppc64el.deb
sudo dpkg -i cuda-repo-ubuntu1804-11-1-local_11.1.0-455.23.05-1_ppc64el.deb
sudo apt-key add /var/cuda-repo-ubuntu1804-11-1-local/7fa2af80.pub
sudo apt-get update
sudo apt-get -y install cuda
```
## Install nvidia-container-runtime
```
apt -y install curl
curl -s -L https://nvidia.github.io/nvidia-container-runtime/gpgkey | \
  sudo apt-key add -
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-container-runtime/$distribution/nvidia-container-runtime.list | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-runtime.list
sudo apt-get update
```
```
sudo apt-get install nvidia-container-runtime
```
## Install docker
```
sudo apt-get remove docker docker-engine docker-ce docker-ce-cli docker.io containerd runc
sudo apt-get update && sudo apt-get install containerd
```
```
wget https://oplab9.parqtec.unicamp.br/pub/ppc64el/docker/version-19.03.8/ubuntu-bionic/docker-ce_19.03.8~3-0~ubuntu-bionic_ppc64el.deb
wget https://oplab9.parqtec.unicamp.br/pub/ppc64el/docker/version-19.03.8/ubuntu-bionic/docker-ce-cli_19.03.8~3-0~ubuntu-bionic_ppc64el.deb
```
```
sudo dpkg -i docker-ce-cli_19.03.8~3-0~ubuntu-bionic_ppc64el.deb docker-ce_19.03.8~3-0~ubuntu-bionic_ppc64el.deb
```

## Test docker
```
docker run -it --rm ubuntu:xenial bash
docker run -it --rm ibmcom/tensorflow-ppc64le bash
docker run -it --rm --gpus all ubuntu nvidia-smi
docker run -it --rm --gpus all ibmcom/tensorflow-ppc64le:latest-gpu-py3 python
docker images
```
## Build docker images - cpu
```
apt -y install git
git clone https://github.com/anji993/build.git
cd build
git checkout anji993-patch-1
cd ppc64le_builds/images/cpu
cp Dockerfile.manylinux_2014 Dockerfile
docker build -t manylinux_2014 .
```
## Build docker images - gpu
```
cd
cd build
cd ppc64le_builds/images/gpu
cp Dockerfile.manylinux_2014.cuda10_1 Dockerfile
docker build -t manylinux_2014.cuda10_1 .
```
## Build tensorflow - cpu
```
docker run -it manylinux_2014 /bin/bash
cd
```
```
git clone https://github.com/tensorflow/tensorflow.git
cd tensorflow
git pull --tags
git checkout v2.3.2
```
```
set -e
rm -rf ./tensorflow_pkg
```
```
cat <<EOT > .tf_configure.bazelrc
build --action_env PYTHON_BIN_PATH="/opt/python/cp36-cp36m/bin/python"
build --action_env PYTHON_LIB_PATH="/opt/python/cp36-cp36m/lib/python3.6/site-packages"
build --python_path="/opt/python/cp36-cp36m/bin/python"
build:xla --define with_xla_support=true
build --config=xla
build:opt --copt=-mcpu=power8
build:opt --copt=-mtune=power8
build:opt --define with_default_optimizations=true
test --flaky_test_attempts=3
test --test_size_filters=small,medium
test:v1 --test_tag_filters=-benchmark-test,-no_oss,-gpu,-oss_serial
test:v1 --build_tag_filters=-benchmark-test,-no_oss,-gpu
test:v2 --test_tag_filters=-benchmark-test,-no_oss,-gpu,-oss_serial,-v1only
test:v2 --build_tag_filters=-benchmark-test,-no_oss,-gpu,-v1only
build --action_env TF_CONFIGURE_IOS="0"
EOT
```
```
cat <<EOT > ./tools/python_bin_path.sh
export PYTHON_BIN_PATH=/opt/python/cp36-cp36m/bin/python
EOT
```
```
BAZEL_LINKLIBS=-l%:libstdc++.a bazel build -c opt --config=v2 --local_resources 4096,4.0,1.0 \
    //tensorflow/tools/pip_package:build_pip_package
```
```
BAZEL_LINKLIBS=-l%:libstdc++.a bazel build -c opt --config=v2 --local_ram_resources=4096 --local_cpu_resources=8 \
    //tensorflow/tools/pip_package:build_pip_package
```
```
BAZEL_LINKLIBS=-l%:libstdc++.a bazel build -c opt --config=v2 \
    //tensorflow/tools/pip_package:build_pip_package
```
```
bazel-bin/tensorflow/tools/pip_package/build_pip_package --cpu ./tensorflow_pkg
```
## Build tensorflow - gpu
```
docker run -it manylinux_2014.cuda10_1 /bin/bash
cd
ln -s /usr/local/lib/libcrypt.so.2 /usr/lib64/libcrypt.so.2
```
```
git clone https://github.com/tensorflow/tensorflow.git
cd tensorflow
git pull --tags
git checkout v2.3.2
```
```
set -e
rm -rf ./tensorflow_pkg
```
```
cat <<EOT > .tf_configure.bazelrc
build --action_env PYTHON_BIN_PATH="/opt/python/cp37-cp37m/bin/python"
build --action_env PYTHON_LIB_PATH="/opt/python/cp37-cp37m/lib/python3.7/site-packages"
build --python_path="/opt/python/cp37-cp37m/bin/python"
build:xla --define with_xla_support=true
build --config=xla
build --action_env TF_CUDA_PATHS="/usr,/usr/lib64,/usr/local/cuda,/usr/local/cuda-10.1/targets/ppc64le-linux/lib,/usr/local/cuda-10.2/targets/ppc64le-linux"
build --action_env CUDA_TOOLKIT_PATH="/usr/local/cuda"
build --action_env TF_CUDA_COMPUTE_CAPABILITIES="6.0"
build --action_env LD_LIBRARY_PATH="/usr/local/nvidia/lib:/usr/local/nvidia/lib64"
build --action_env GCC_HOST_COMPILER_PATH="/opt/rh/devtoolset-8/root/usr/bin/gcc"
build --config=cuda
build:opt --copt=-mcpu=power8
build:opt --copt=-mtune=power8
build:opt --define with_default_optimizations=true
test --flaky_test_attempts=3
test --test_size_filters=small,medium
test:v1 --test_tag_filters=-benchmark-test,-no_oss,-no_gpu,-oss_serial
test:v1 --build_tag_filters=-benchmark-test,-no_oss,-no_gpu
test:v2 --test_tag_filters=-benchmark-test,-no_oss,-no_gpu,-oss_serial,-v1only
test:v2 --build_tag_filters=-benchmark-test,-no_oss,-no_gpu,-v1only
build --action_env TF_CONFIGURE_IOS="0"
EOT
```
```
cat <<EOT > ./tools/python_bin_path.sh
export PYTHON_BIN_PATH=/opt/python/cp37-cp37m/bin/python
EOT
```
```
bazel --host_jvm_args="-Xms512m" --host_jvm_args="-Xmx4096m" build -c opt --config=v2 --local_ram_resources=8192 --local_cpu_resources=16 //tensorflow/tools/pip_package:build_pip_package
```
```
bazel-bin/tensorflow/tools/pip_package/build_pip_package ./tensorflow_pkg
```

