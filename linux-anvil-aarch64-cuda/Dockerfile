ARG CUDA_VER
ARG CENTOS_VER
FROM --platform=linux/arm64 nvidia/cuda:${CUDA_VER}-devel-centos${CENTOS_VER}

LABEL maintainer="conda-forge <conda-forge@googlegroups.com>"

ADD qemu-aarch64-static /usr/bin/qemu-aarch64-static

# Set CUDA_VER during runtime.
ARG CUDA_VER
ARG CENTOS_VER
ENV CUDA_VER=${CUDA_VER} \
    CENTOS_VER=${CENTOS_VER}

# Set an encoding to make things work smoothly.
ENV LANG en_US.UTF-8
ENV LANGUAGE=en_US.UTF-8

# Set path to CUDA install.
ENV CUDA_HOME /usr/local/cuda

# we want to persist a path in ldconfig (to avoid having to always set LD_LIBRARY_PATH), but *after* the existing entries;
# since entries in ld.so.conf.d have precedence before the preconfigured directories, we first add the latter to the former
# the upstream images for 10.x all have libcuda.so under $CUDA_HOME/compat;
# add this to the ldconfig so it will be found correctly.
# don't forget to update settings by running ldconfig
RUN ldconfig -v 2>/dev/null | grep -v ^$'\t' | cut -f1 -d":" >> /etc/ld.so.conf.d/cuda-$CUDA_VER.conf && \
    echo "$CUDA_HOME/compat" >> /etc/ld.so.conf.d/cuda-$CUDA_VER.conf && \
    ldconfig

# bust the docker cache so that we always rerun the installs below
ADD https://loripsum.net/api /opt/docker/etc/gibberish

# Resolves a nasty NOKEY warning that appears when using yum.
# Naming convention changed with cos8 - see:
# * https://lists.centos.org/pipermail/centos-devel/2019-September/017847.html
# * https://www.centos.org/keys/#project-keys
RUN if [ "$CENTOS_VER" -le "7" ]; then \
        rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-${CENTOS_VER} && \
        rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-${CENTOS_VER}-aarch64; \
    else \
        rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-centosofficial; \
    fi

# Remove preinclude system compilers
RUN rpm -e --nodeps --verbose gcc gcc-c++

# Install basic requirements.
COPY scripts/yum_clean_all /opt/docker/bin/
RUN yum update -y --disablerepo=cuda && \
    yum install -y \
        bzip2 \
        sudo \
        tar \
        which \
        && \
    /opt/docker/bin/yum_clean_all

# Fix locale in CentOS8 images
# See https://github.com/CentOS/sig-cloud-instance-images/issues/154
RUN yum install -y glibc-langpack-en && \
    /opt/docker/bin/yum_clean_all

# Run common commands
COPY scripts/run_commands /opt/docker/bin/run_commands
RUN /opt/docker/bin/run_commands

# Download and cache new compiler packages.
# Should speedup installation of them on CIs.
RUN source /opt/conda/etc/profile.d/conda.sh && \
    conda activate && \
    conda create -n test --yes --quiet --download-only \
        conda-forge::binutils_impl_linux-aarch64 \
        conda-forge::binutils_linux-aarch64 \
        conda-forge::gcc_impl_linux-aarch64 \
        conda-forge::gcc_linux-aarch64 \
        conda-forge::gfortran_impl_linux-aarch64 \
        conda-forge::gfortran_linux-aarch64 \
        conda-forge::gxx_impl_linux-aarch64 \
        conda-forge::gxx_linux-aarch64 \
        conda-forge::libgcc-ng \
        conda-forge::libgfortran-ng \
        conda-forge::libstdcxx-ng && \
    conda remove --yes --quiet -n test --all && \
    conda clean -tiy && \
    chgrp -R lucky /opt/conda && \
    chmod -R g=u /opt/conda

# Download and cache CUDA related packages.
RUN source /opt/conda/etc/profile.d/conda.sh && \
    conda activate && \
    conda create -n test --yes --quiet --download-only \
        conda-forge::cudatoolkit=${CUDA_VER} \
        && \
    conda remove --yes --quiet -n test --all && \
    conda clean -tiy && \
    chgrp -R lucky /opt/conda && \
    chmod -R g=u /opt/conda

# Add a file for users to source to activate the `conda`
# environment `root`. Also add a file that wraps that for
# use with the `ENTRYPOINT`.
COPY linux-anvil-aarch64/entrypoint_source /opt/docker/bin/entrypoint_source
COPY scripts/entrypoint /opt/docker/bin/entrypoint

# Ensure that all containers start with tini and the user selected process.
# Activate the `conda` environment `root`.
# Provide a default command (`bash`), which will start if the user doesn't specify one.
ENTRYPOINT [ "/opt/conda/bin/tini", "--", "/opt/docker/bin/entrypoint" ]
CMD [ "/bin/bash" ]
