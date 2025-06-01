ARG BUILD_FROM=ghcr.io/home-assistant/amd64-base:3.21
FROM $BUILD_FROM as builder

ARG RTLSDRBLOG_REPO="https://github.com/rtlsdrblog/rtl-sdr-blog.git"

ENV LANG C.UTF-8
ENV LC_ALL C.UTF-8


# Copied with minor edits from https://github.com/hertzg/rtl_433_docker/blob/master/images/alpine/build-context/Dockerfile
RUN apk add --no-cache --virtual .buildDeps \
    build-base \
	libusb-1.0-0-dev \
    hackrf \
    hackrf-dev \
    librtlsdr-dev \
    soapy-sdr \
    soapy-sdr-dev \
    openssl-dev \
    cmake \
    git

WORKDIR /tmp/build_drivers

# Clone, build, and install rtl-sdr-blog drivers
RUN git clone --depth 1 ${RTLSDRBLOG_REPO} . && \
    mkdir build && \
    cd build && \
    # Using -DDETACH_KERNEL_DRIVER=ON to help if kernel drivers are loaded
    # Using -DINSTALL_UDEV_RULES=ON to install udev rules for device access
    cmake ../ -DINSTALL_UDEV_RULES=ON -DDETACH_KERNEL_DRIVER=ON && \
    make -j"$(nproc)" && \
    make install && \
    ldconfig && \
    # Clean up driver build directory
    cd / && \
    rm -rf /tmp/build_drivers

WORKDIR /build

FROM builder as rtl_433_builder

RUN git clone https://github.com/merbanan/rtl_433
WORKDIR ./rtl_433

# Build a specific commit or tag.
ARG rtl433GitRevision=25.02
RUN git checkout ${rtl433GitRevision}
WORKDIR ./build
RUN cmake ..
RUN make -j 4

WORKDIR /build/root
WORKDIR /build/rtl_433/build
RUN make DESTDIR=/build/root/ install

FROM builder as soapy_hackrf_builder
RUN git clone https://github.com/pothosware/SoapyHackRF.git
WORKDIR SoapyHackRF
RUN git checkout 6c0c33f0aa44c3080674e6bca0273184d3e9eb44
WORKDIR build
RUN cmake ..
RUN make -j4
RUN make install

FROM $BUILD_FROM

ENV LANG C.UTF-8

RUN apk add --no-cache libusb \
    hackrf \
    librtlsdr \
    soapy-sdr \
    sed
WORKDIR /root
COPY --from=rtl_433_builder /build/root/ /

COPY --from=soapy_hackrf_builder /usr/lib /lib-s
RUN mkdir -p /usr/local/lib/SoapySDR/modules0.8/
COPY --from=soapy_hackrf_builder /usr/local/lib/SoapySDR/modules0.8/libHackRFSupport.so /usr/local/lib/SoapySDR/modules0.8/

# Run script
COPY run.sh /
RUN chmod a+x /run.sh

CMD [ "/run.sh" ]
