FROM nvcr.io/nvidia/tensorflow:22.12-tf1-py3

ARG EFA_INSTALLER_VERSION=latest
ARG AWS_OFI_NCCL_VERSION=aws
ARG NCCL_REPO=koyongse
ARG NCCL_VER=dynamic-buffer-depth
ARG OPAL_PREFIX=""

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

RUN apt-get update -y && \
     DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
         apt-transport-https \
         ca-certificates \
         gnupg \
         wget \
         openssh-client \
         openssh-server \
         build-essential \
         autoconf \
         gdb \
         automake \
         cmake \
         pciutils \
         tcl \
         environment-modules \
         rdma-core \
         libtool && \
     mkdir -p /var/run/sshd && \
     sed -I 's/[ #]\(.*StrictHostKeyChecking \).*/ \1no/g' /etc/ssh/ssh_config && \
     echo "    UserKnownHostsFile /dev/null" >> /etc/ssh/ssh_config && \
     sed -I 's/#\(StrictModes \).*/\1no/g' /etc/ssh/sshd_config

# Install Nsight-systems
RUN wget -qO - https://developer.download.nvidia.com/devtools/repos/ubuntu2004/amd64/nvidia.pub | apt-key add - && \
       echo "deb https://developer.download.nvidia.com/devtools/repos/ubuntu2004/amd64/ /" >> /etc/apt/sources.list.d/nsight.list && \
    apt-get update -y && \
      DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
         nsight-systems-2023.1.1 && \
    ln -s -f /opt/nvidia/nsight-systems/2023.1.1/bin/nsys /usr/local/cuda/bin/nsys

RUN python -m pip install --no-cache-dir git+https://github.com/huggingface/transformers
RUN python -m pip install --no-cache-dir datasets evaluate toposort networkx pytest nltk tqdm progressbar pynvml wget
RUN python -m pip install --no-cache-dir git+https://github.com/NVIDIA/dllogger

# Install OpenMPI
RUN mkdir -p /tmp/openmpi
WORKDIR /tmp/openmpi
RUN wget -q "https://download.open-mpi.org/release/open-mpi/v4.1/openmpi-4.1.4.tar.gz" && \
    gunzip -c openmpi-4.1.4.tar.gz | tar xf -

# Install EFA
WORKDIR /tmp
RUN wget -q "https://efa-installer.amazonaws.com/aws-efa-installer-${EFA_INSTALLER_VERSION}.tar.gz" && \
    tar -xf "aws-efa-installer-${EFA_INSTALLER_VERSION}.tar.gz"
WORKDIR /tmp/aws-efa-installer
RUN ./efa_installer.sh -y -g -d --skip-kmod --skip-limit-conf --no-verify

# Install NCCL
RUN git clone https://github.com/${NCCL_REPO}/nccl /opt/nccl
WORKDIR /opt/nccl
RUN git checkout ${NCCL_VER} && \
    make -j src.build CUDA_HOME=/usr/local/cuda \
      NVCC_GENCODE="-gencode=arch=compute_86,code=sm_86 -gencode=arch=compute_80,code=sm_80 -gencode=arch=compute_75,code=sm_75 -gencode=arch=compute_70,code=sm_70 -gencode=arch=compute_60,code=sm_60"

# Install AWS-OFI-NCCL plugin
RUN git clone https://github.com/aws/aws-ofi-nccl.git /opt/aws-ofi-nccl
WORKDIR /opt/aws-ofi-nccl
RUN git checkout ${AWS_OFI_NCCL_VERSION} && \
    ./autogen.sh && \
    ./configure --prefix=/opt/aws-ofi-nccl/install \
         --with-libfabric=/opt/amazon/efa/ \
         --with-cuda=/usr/local/cuda \
         --with-nccl=/opt/nccl/build \
         --with-mpi=/usr/local/mpi/ && \
    make && \
    make install

# Build MPI
ARG LD_PRELOAD=/opt/nccl/build/lib/libnccl.so:/opt/aws-ofi-nccl/install/lib/libnccl-net.so
WORKDIR /tmp/openmpi/openmpi-4.1.4
RUN ./configure --prefix=/shared --with-libfabric=/opt/amazon/efa --with-sge --with-hwloc=internal; make all install


RUN git clone https://github.com/NVIDIA/DeepLearningExamples.git && \
    cp -R DeepLearningExamples/TensorFlow/LanguageModeling/BERT /workspace/

CMD ["/bin/bash", "--login"]
