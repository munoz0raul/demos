# docker run -it --rm  --entrypoint /bin/bash everest-core
# syntax=docker/dockerfile:1
FROM debian:11.0 AS builder

ENV REPO="https://github.com/EVerest/everest-core.git"
ENV BRANCH="main"
ENV EVEREST_CONFIG="config-sil.yaml"
ENV OCPP_CONFIG="ocpp_user_config.json"

RUN apt-get update \
    && apt-get install --no-install-recommends -y \
    git \
    rsync \
    wget \
    cmake \
    doxygen \
    graphviz \
    build-essential \
    clang-format \
    clang-tidy \
    cppcheck \
    libboost-all-dev \
    maven \
    openjdk-11-jdk \
    nodejs \
    npm \
    libsqlite3-dev \
    python3-pip \
    libssl-dev \
    libcurl4-openssl-dev \
    libpcap-dev \
    libtool \
    automake \
    autoconf \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /workspace/

RUN git clone https://git.ti.com/cgit/rpmsg/ti-rpmsg-char

WORKDIR /workspace/ti-rpmsg-char

RUN git checkout adb964ee72f99545e2ad55c50cfd14015a7d837e
RUN autoreconf -i
RUN ./configure
RUN make
RUN make install

WORKDIR /workspace/everest

# to avoid caching
ARG BUILD_DATE=Unknown 

# add github to known hosts
RUN mkdir ~/.ssh
RUN ssh-keyscan github.com > ~/.ssh/known_hosts

RUN mkdir -p /workspace/everest/cpm_source_cache    
ENV CPM_SOURCE_CACHE="/workspace/everest/cpm_source_cache"

RUN git clone https://github.com/EVerest/everest-cmake.git
RUN git clone https://github.com/EVerest/everest-utils.git
WORKDIR /workspace/everest/everest-utils/ev-dev-tools
RUN python3 -m pip install .
WORKDIR /workspace/everest
RUN git clone https://github.com/EVerest/everest-dev-environment.git
WORKDIR /workspace/everest/everest-dev-environment/dependency_manager
RUN python3 -m pip install .
WORKDIR /workspace/everest
RUN git clone https://github.com/EVerest/ext-switchev-iso15118.git

WORKDIR /workspace/everest

RUN rm -rf "/workspace/everest/everest-core"
RUN git clone ${REPO}

RUN git clone https://github.com/PionixPublic/ti-am62x-evse-sdk.git

RUN cp -r ti-am62x-evse-sdk/everest-module/modules/TiAM625SDK/ everest-core/modules/
RUN rm -r everest-core/modules/TiAM625SDK/pb_codec
RUN cp -r ti-am62x-evse-sdk/firmware/pb_codec/ everest-core/modules/TiAM625SDK/
RUN echo "ev_add_cpp_module(TiAM625SDK)" >> everest-core/modules/CMakeLists.txt 

RUN rm -rf "/workspace/everest/everest-core/build"
WORKDIR /workspace/everest/everest-core
RUN git checkout "${BRANCH}"
RUN mkdir /workspace/everest/everest-core/build
WORKDIR /workspace/everest/everest-core/build 
RUN cmake .. -DEVEREST_BUILD_ALL_MODULES=ON -DCMAKE_INSTALL_PREFIX=/opt/everest
RUN make -j"$(nproc)" install

RUN mkdir -p /opt/everest/config/user-config
COPY "${EVEREST_CONFIG}" /opt/everest/config/
COPY "${OCPP_CONFIG}" /opt/everest/config/
RUN if [ "${EVEREST_CONFIG}" != "config.yaml" ]; then mv /opt/everest/config/"${EVEREST_CONFIG}" /opt/everest/config/config.yaml ; fi
RUN if [ "${OCPP_CONFIG}" != "ocpp-config.json" ]; then mv /opt/everest/config/"${OCPP_CONFIG}" /opt/everest/config/ocpp-config.json ; fi

COPY logging.ini /opt/everest/config

# syntax=docker/dockerfile:1
FROM debian:11.0-slim

RUN apt-get update \
    && apt-get install --no-install-recommends -y \
    openjdk-11-jre \
    nodejs \
    npm \
    python3-pip \
    sqlite3 \
    libboost-program-options1.74.0 \
    libboost-log1.74.0 \
    libboost-chrono1.74.0 \
    libboost-system1.74.0 \
    libssl1.1 \
    libcurl4 \
    less \
    gcc \
    python3-dev \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

COPY --from=builder /workspace/everest/ext-switchev-iso15118/requirements.txt ./
RUN pip install --user -r requirements.txt

WORKDIR /opt/everest
COPY --from=builder /opt/everest ./

CMD [ "/opt/everest/bin/manager", "--conf", "/opt/everest/config/config.yaml" ]
