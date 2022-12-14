# Copyright 2021-2022 The MathWorks, Inc.
# Builds Docker image with 
# 1. MATLAB - Using MPM
# 2. MATLAB Integration for Jupyter (VNC)
# on a base image of jupyter/base-notebook.

## Sample Build Command:
# docker build --build-arg MATLAB_RELEASE=r2022a \
#              --build-arg MATLAB_PRODUCT_LIST="MATLAB Deep_Learning_Toolbox Symbolic_Math_Toolbox"\
#              --build-arg LICENSE_SERVER=12345@hostname.com \
#              -t my_matlab_image_name .

# Specify release of MATLAB to build. (use lowercase, default is r2022a)
ARG MATLAB_RELEASE=r2022a

# Specify the list of products to install into MATLAB, 
ARG MATLAB_PRODUCT_LIST="MATLAB"

# Optional Network License Server information
ARG LICENSE_SERVER

# If LICENSE_SERVER is provided then SHOULD_USE_LICENSE_SERVER will be set to "_use_lm"
ARG SHOULD_USE_LICENSE_SERVER=${LICENSE_SERVER:+"_with_lm"}

# Default DDUX information
ARG MW_CONTEXT_TAGS=MATLAB_PROXY:JUPYTER:MPM:V1

# Base Jupyter image without LICENSE_SERVER, using python-3.9.12 as websockify only supports upto 3.9
FROM jupyter/base-notebook:python-3.9.12 AS base_jupyter_image

# Base Jupyter image with LICENSE_SERVER
FROM jupyter/base-notebook:python-3.9.12 AS base_jupyter_image_with_lm
ARG LICENSE_SERVER
# If license server information is available, then use it to set environment variable
ENV MLM_LICENSE_FILE=${LICENSE_SERVER}

# Select base Jupyter image based on whether LICENSE_SERVER is provided
FROM base_jupyter_image${SHOULD_USE_LICENSE_SERVER}
ARG MW_CONTEXT_TAGS
ARG MATLAB_RELEASE
ARG MATLAB_PRODUCT_LIST

# Switch to root user
USER root
ENV DEBIAN_FRONTEND="noninteractive" TZ="Etc/UTC"

## Installing Dependencies for Ubuntu 20.04
# For MATLAB : Get base-dependencies.txt from matlab-deps repository on GitHub
# For mpm : wget, unzip, ca-certificates
# For MATLAB Integration for Jupyter (VNC): xvfb dbus-x11 firefox xfce4 xfce4-panel xfce4-session xfce4-settings xorg xubuntu-icon-theme curl xscreensaver

# List of MATLAB Dependencies for Ubuntu 20.04 and specified MATLAB_RELEASE
ARG MATLAB_DEPS_REQUIREMENTS_FILE="https://raw.githubusercontent.com/mathworks-ref-arch/container-images/main/matlab-deps/${MATLAB_RELEASE}/ubuntu20.04/base-dependencies.txt"
ARG MATLAB_DEPS_REQUIREMENTS_FILE_NAME="matlab-deps-${MATLAB_RELEASE}-base-dependencies.txt"

# Install dependencies
RUN wget ${MATLAB_DEPS_REQUIREMENTS_FILE} -O ${MATLAB_DEPS_REQUIREMENTS_FILE_NAME} && \
    export DEBIAN_FRONTEND=noninteractive && apt-get update && \
    xargs -a ${MATLAB_DEPS_REQUIREMENTS_FILE_NAME} -r apt-get install --no-install-recommends -y \
    dbus-x11 \
    firefox \
    xfce4 \
    xfce4-panel \
    xfce4-session \
    xfce4-settings \
    xorg \
    xubuntu-icon-theme \
    curl \
    xscreensaver \
    qt5-default \
    mesa-utils \
    pciutils \
    xserver-xorg-core \
    libxv1 \
    x11-xserver-utils \
    && apt-get remove -y gnome-screensaver  \
    && apt-get clean \
    && apt-get -y autoremove \
    && rm -rf /var/lib/apt/lists/*

# Run mpm to install MATLAB in the target location and delete the mpm installation afterwards
RUN wget -q https://www.mathworks.com/mpm/glnxa64/mpm && \ 
    chmod +x mpm && \
    ./mpm install \
    --release=${MATLAB_RELEASE} \
    --destination=/opt/matlab \
    --products ${MATLAB_PRODUCT_LIST} && \
    rm -f mpm /tmp/mathworks_root.log && \
    ln -s /opt/matlab/bin/matlab /usr/local/bin/matlab

# Install patched glibc - See https://github.com/mathworks/build-glibc-bz-19329-patch
WORKDIR /packages
RUN export DEBIAN_FRONTEND=noninteractive && \
    apt-get update && apt-get clean && apt-get autoremove && \
    wget -q https://github.com/mathworks/build-glibc-bz-19329-patch/releases/download/ubuntu-focal/all-packages.tar.gz && \
    tar -x -f all-packages.tar.gz \
    --exclude glibc-*.deb \
    --exclude libc6-dbg*.deb && \
    apt-get install --yes --no-install-recommends --allow-downgrades ./*.deb && \
    rm -fr /packages
WORKDIR /

# Install tigervnc to /usr/local
RUN curl -sSfL 'https://sourceforge.net/projects/tigervnc/files/stable/1.10.1/tigervnc-1.10.1.x86_64.tar.gz/download' \
    | tar -zxf - -C /usr/local --strip=2

# noVNC provides VNC over browser capability
# Set default install location for noVNC
ARG NOVNC_PATH=/opt/noVNC

# Get noVNC
RUN mkdir -p ${NOVNC_PATH} \
    && curl -sSfL 'https://github.com/novnc/noVNC/archive/v1.2.0.tar.gz' \
    | tar -zxf - -C ${NOVNC_PATH} --strip=1 \
    && chown -R ${NB_USER}:users ${NOVNC_PATH}

# Install paraview
WORKDIR /opt

RUN wget -O ParaView-5.10.1.tar.gz "https://www.paraview.org/paraview-downloads/download.php?submit=Download&version=v5.10&type=binary&os=Linux&downloadFile=ParaView-5.10.1-MPI-Linux-Python3.9-x86_64.tar.gz"

RUN tar -xzvf ParaView-5.10.1.tar.gz && \
    rm ParaView-5.10.1.tar.gz
    
ENV PATH=$PATH:/opt/ParaView-5.10.1-MPI-Linux-Python3.9-x86_64/bin

ENV LIBGL_ALWAYS_INDIRECT=y

ENV LIBGL_DEBUG_PLUGINS=y

WORKDIR /tmp
RUN curl -L -o virtualgl_2.5.2_amd64.deb 'https://downloads.sourceforge.net/project/virtualgl/2.5.2/virtualgl_2.5.2_amd64.deb?r=https%3A%2F%2Fsourceforge.net%2Fprojects%2Fvirtualgl%2Ffiles%2F2.5.2&ts=1509495317&use_mirror=auto'
RUN dpkg -i virtualgl_2.5.2_amd64.deb
RUN printf "1\nn\nn\nn\nx\n" | /opt/VirtualGL/bin/vglserver_config

WORKDIR /

# JOVYAN is the default user in jupyter/base-notebook.
# JOVYAN is being set to be passwordless. 
# This allows users to easily wake the desktop when it goes to sleep.
RUN passwd $NB_USER -d

# Change user to jovyan from root as we do not want any changes to be made as root in the container
USER $NB_USER

# Get websockify
RUN conda install -y -q websockify=0.9.0

# Set environment variable for python package jupyter-matlab-vnc-proxy
ENV NOVNC_PATH=${NOVNC_PATH}

# Pip install the latest version of the integration
RUN curl -s https://api.github.com/repos/mathworks/jupyter-matlab-vnc-proxy/releases/latest | grep tarball_url | cut -d '"' -f 4 | xargs python -m pip install

# Ensure jupyter-server-proxy JupyterLab extension is installed
RUN jupyter labextension install @jupyterlab/server-proxy

# Move MATLAB resource files to the expected locations
RUN export RESOURCES_LOC=$(python -c "import jupyter_matlab_vnc_proxy as pkg; print(pkg.__path__[0])")/resources \
    && mkdir -p ${HOME}/.local/share/applications ${HOME}/Desktop ${HOME}/.local/share/ ${HOME}/.icons \
    && cp ${RESOURCES_LOC}/MATLAB.desktop ${HOME}/Desktop/ \
    && cp ${RESOURCES_LOC}/MATLAB.desktop ${HOME}/.local/share/applications\
    && ln -s ${RESOURCES_LOC}/matlab_icon.png ${HOME}/.icons/matlab_icon.png \
    && cp ${RESOURCES_LOC}/matlab_launcher.py ${HOME}/.local/share/ \
    && cp ${RESOURCES_LOC}/mw_lite.html ${NOVNC_PATH}

# Fixes occasional failure to start VNC desktop, which requires a reloading of the webpage to fix.
RUN touch ${HOME}/.Xauthority

