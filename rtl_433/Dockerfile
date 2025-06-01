ARG BUILD_FROM=ghcr.io/home-assistant/amd64-base:3.21
FROM $BUILD_FROM as builder
MAINTAINER pbkhrv@pm.me

ENV LANG C.UTF-8

# Copied with minor edits from https://github.com/hertzg/rtl_433_docker/blob/master/images/alpine/build-context/Dockerfile
RUN apk add --no-cache --virtual .buildDeps \
    build-base \
    libusb-dev \
    hackrf \
    hackrf-dev \
    soapy-sdr \
    soapy-sdr-dev \
    openssl-dev \
    cmake \
    git

WORKDIR /build

# Build RTL-SDR Blog drivers with bias tee support
FROM builder as rtlsdr_blog_builder
RUN git clone https://github.com/rtlsdrblog/rtl-sdr-blog.git
WORKDIR rtl-sdr-blog
RUN mkdir build
WORKDIR build
RUN cmake ../ -DINSTALL_UDEV_RULES=ON -DDETACH_KERNEL_DRIVER=ON
RUN make -j 4
RUN make DESTDIR=/build/rtlsdr-root/ install

FROM builder as rtl_433_builder

# Copy RTL-SDR Blog drivers for rtl_433 build
COPY --from=rtlsdr_blog_builder /build/rtlsdr-root/ /
RUN ldconfig
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

# Install RTL-SDR Blog drivers
COPY --from=rtlsdr_blog_builder /build/rtlsdr-root/ /

COPY --from=rtl_433_builder /build/root/ /

COPY --from=soapy_hackrf_builder /usr/lib /lib-s
RUN mkdir -p /usr/local/lib/SoapySDR/modules0.8/
COPY --from=soapy_hackrf_builder /usr/local/lib/SoapySDR/modules0.8/libHackRFSupport.so /usr/local/lib/SoapySDR/modules0.8/

RUN ldconfig


# Run script
COPY run.sh /
RUN chmod a+x /run.sh

CMD [ "/run.sh" ]