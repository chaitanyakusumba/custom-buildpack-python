ARG UBUNTU_VERSION=18.04

ARG ARCH=
ARG CUDA=11.2
FROM nvidia/cuda${ARCH:+-$ARCH}:${CUDA}.1-base-ubuntu${UBUNTU_VERSION} as base
# ARCH and CUDA are specified again because the FROM directive resets ARGs
# (but their default value is retained if set previously)
ARG ARCH
ARG CUDA
ARG CUDNN=8.1.0.77-1
ARG CUDNN_MAJOR_VERSION=8
ARG LIB_DIR_PREFIX=x86_64
ARG LIBNVINFER=7.2.2-1
ARG LIBNVINFER_MAJOR_VERSION=7

# Needed for string substitution
SHELL ["/bin/bash", "-c"]
# Pick up some TF dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
        build-essential \
        cuda-command-line-tools-${CUDA/./-} \
        libcublas-${CUDA/./-} \
        cuda-nvrtc-${CUDA/./-} \
        libcufft-${CUDA/./-} \
        libcurand-${CUDA/./-} \
        libcusolver-${CUDA/./-} \
        libcusparse-${CUDA/./-} \
        curl \
        libcudnn8=${CUDNN}+cuda${CUDA} \
        libfreetype6-dev \
        libhdf5-serial-dev \
        libzmq3-dev \
        pkg-config \
        software-properties-common \
        unzip

# Install TensorRT if not building for PowerPC
# NOTE: libnvinfer uses cuda11.1 versions
RUN [[ "${ARCH}" = "ppc64le" ]] || { apt-get update && \
        apt-get install -y --no-install-recommends libnvinfer${LIBNVINFER_MAJOR_VERSION}=${LIBNVINFER}+cuda11.1 \
        libnvinfer-plugin${LIBNVINFER_MAJOR_VERSION}=${LIBNVINFER}+cuda11.1 \
        && apt-get clean \
        && rm -rf /var/lib/apt/lists/*; }

# For CUDA profiling, TensorFlow requires CUPTI.
ENV LD_LIBRARY_PATH /usr/local/cuda/extras/CUPTI/lib64:/usr/local/cuda/lib64:/usr/local/cuda-11.1/lib64:$LD_LIBRARY_PATH

# Link the libcuda stub to the location where tensorflow is searching for it and reconfigure
# dynamic linker run-time bindings
RUN ln -s /usr/local/cuda/lib64/stubs/libcuda.so /usr/local/cuda/lib64/stubs/libcuda.so.1 \
    && echo "/usr/local/cuda/lib64/stubs" > /etc/ld.so.conf.d/z-cuda-stubs.conf \
    && ldconfig

# See http://bugs.python.org/issue19846
ENV LANG C.UTF-8

RUN apt-get update && apt-get install -y \
    python3 \
    python3-pip

RUN python3 -m pip --no-cache-dir install --upgrade \
    "pip<20.3" \
    setuptools

# Some TF tools expect a "python" binary
RUN ln -s $(which python3) /usr/local/bin/python

ARG TF_PACKAGE=tensorflow
ARG TF_PACKAGE_VERSION=

ENV CNB_STACK_ID="com.digitaldots.stacks.python"
LABEL io.buildpacks.stack.id="com.digitaldots.stacks.python"

# Create CNB user and group
ENV CNB_USER_ID=1000
ENV CNB_GROUP_ID=1000
RUN groupadd cnb --gid ${CNB_GROUP_ID} && \
  useradd --uid ${CNB_USER_ID} --gid ${CNB_GROUP_ID} -m -s /bin/bash cnb
RUN apt-get update && \
  apt-get install -y xz-utils ca-certificates && \
  rm -rf /var/lib/apt/lists/*

#=========================================#####################
## ---Build image---

FROM base as build

# Install packages that we want to make available at build time
RUN apt-get update && \
  apt-get install -y git wget jq && \
  rm -rf /var/lib/apt/lists/* && \
  wget https://github.com/sclevine/yj/releases/download/v5.0.0/yj-linux -O /usr/local/bin/yj && \
  chmod +x /usr/local/bin/yj

# Declare dependencies this image provides
LABEL io.buildpacks.stack.mixins="[\"build:wget\", \"build:jq\", \"build:yj\"]"

# Set user and group (as declared in base image)
USER ${CNB_USER_ID}:${CNB_GROUP_ID}

## ---Run image---

FROM base as run

# Set user and group (as declared in base image)
USER ${CNB_USER_ID}:${CNB_GROUP_ID}
WORKDIR /service
ADD . /service
WORKDIR /service
# RUN pip3 install -r requirements.txt
ENV LD_LIBRARY_PATH=/usr/local/cuda-11.2/targets/x86_64-linux/lib/:$LD_LIBRARY_PATH
EXPOSE 8080
CMD ["python", "main.py"]