FROM centos:6
MAINTAINER BioBuilds Support <support@biobuilds.org>

# Install "lsb_release" script. Also available through the "redhat-lsb-core"
# package, but that would force us to install other things we don't need.
COPY lsb_release /usr/bin/lsb_release

# Make sure a basic set of command line tools are available
RUN yum update -y && yum install -y \
        bzip2 \
        curl \
        file \
        findutils \
        gawk \
        grep \
        gzip \
        net-tools \
        openssl \
        sed \
        tar \
        unzip \
        wget \
        which \
        xz \
    && yum clean all

# Install a basic set of build tools (compilers, make, patch, etc.) from the
# official CentOS repository as well as Continuum Analytics' conda for use as
# a supplemental package manager. (Conda saves us the trouble of having to
# maintain our own tarballs and/or RPM repositories in case we want/need to
# install newer versions of other build tools.)
RUN yum install -y \
        bison \
        chrpath \
        cvs \
        flex \
        gcc \
        gcc-c++ \
        gcc-gfortran \
        git \
        glibc-static \
        libffi-devel \
        libstdc++-static \
        make \
        mercurial \
        nasm \
        patch \
        perl \
        svn \
        vim-common \
    && yum clean all && \
    curl -o /tmp/miniconda.sh \
        "http://download.lab7.io/Miniconda-3.16.0-Linux-x86_64.sh" && \
    /bin/bash /tmp/miniconda.sh -f -b -p /opt/miniconda && \
    # GNU `tar` ignore root's umask by default, so we need to forcibly remove
    # non-root write permissions from the extracted files. Not doing this can
    # cause `conda` to fail in strange ways when running this image with
    # `docker run -u $UID`. In such cases, the user's primary GID will be 0
    # (root), and `conda` assumes that hard links will work if a package's
    # "index.json" has the group write permission set; however, if some file
    # within a package does *NOT* have group write permissions, hard linking
    # will silently fail, leading to an incomplete package installation within
    # the target conda environment.
    chmod -R go-w /opt/miniconda && \
    /opt/miniconda/bin/conda clean --packages --tarballs --source-cache --yes && \
    rm -f /tmp/miniconda.sh
ENV PATH /opt/miniconda/bin:$PATH

# Install "conda-build" from the Anaconda "defaults" channel as well as a
# minimal set of X11 development libraries from the official CentOS repository;
# the X11 libraries are needed for building conda packages that depend on the
# cairo vector graphics library (among other things).
#
# NOTE: Pin conda & conda-build versions to match what's available on ppc64le.
RUN yum install -y \
        libX11-devel libXt-devel libXrender-devel libXext-devel && \
    yum clean all && \
    /opt/miniconda/bin/conda install --yes --quiet \
        'conda==4.0.4' 'conda-build==1.19.2' 'anaconda-client' && \
    echo 'conda ==4.0.4' >> /opt/miniconda/conda-meta/pinned && \
    # See parent (build/centos6) Dockerfile for why this `chmod` is needed
    chmod -R go-w /opt/miniconda && \
    echo 'conda-build ==1.19.2' >> /opt/miniconda/conda-meta/pinned && \
    /opt/miniconda/bin/conda clean --packages --tarballs --source-cache --yes

# Install the CUDA compiler (nvcc). Note that a "functional" libcuda.so (i.e.,
# driver runtime library) is *not* actually needed to build GPU applications as
# the CUDA toolkit contains a stub version which can be used by the linker.
RUN rpm -Uvh "http://developer.download.nvidia.com/compute/cuda/repos/rhel6/x86_64/cuda-repo-rhel6-7.0-28.x86_64.rpm" && \
    yum install -y cuda-minimal-build-7-0 && \
    ln -sfn /usr/local/cuda-7.0 /usr/local/cuda && \
    yum clean all
ENV PATH /usr/local/cuda/bin:$PATH

# Configure things so non-root users can use conda in this container
ENV HOME /tmp
RUN echo 'channels: ["biobuilds", "defaults"]' > /tmp/.condarc && \
    echo 'show_channel_urls: True' >> /tmp/.condarc && \
    echo 'update_dependencies: False' >> /tmp/.condarc && \
    echo 'anaconda_upload: False' >> /tmp/.condarc && \
    chmod 444 /tmp/.condarc

# Entrypoint script so BioBuilds packages can be built by running:
# "docker run -v /path/to/repo:/src lab7io/build:biobuilds <pkg>"
COPY conda-build.sh /
ENTRYPOINT ["/conda-build.sh"]

# Additional build tools present in the build image used for the 2015.11 and
# 2016.04 releases; as of the 2016.11 release, we will replace these with their
# Anaconda equivalents (i.e., as dependencies in the recipe "meta.yaml" files).
#RUN yum install -y \
#        autoconf automake libtool cmake readline-devel readline-static && \
#    yum clean all