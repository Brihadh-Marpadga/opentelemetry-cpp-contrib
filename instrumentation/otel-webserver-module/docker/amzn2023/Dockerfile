FROM amazonlinux:2023

ARG BUILD_NUMBER

LABEL NAME=apm/build-image-webserver-agent-centos6-x64 VERSION=$BUILD_NUMBER

ENV GOSU_ARCH amd64
ENV JDK_ARCH x64

ARG PYTHON_VERSION="2.7.8"
ARG CMAKE_VERSION="3.20"
ARG CMAKE_0VERSION="3.20.0"
ARG GRPC_VERSION="1.70.1"
ARG OPENTELEMETRY_VERSION="1.19.0"
ARG BOOST_VERSION="1.75.0"
ARG BOOST_FILENAME="boost_1_75_0"
ARG APR_VERSION="1.7.0"
ARG EXPAT_VERSION="2.3.0"
ARG EXPAT_RVERSION="R_2_3_0"
ARG APRUTIL_VERSION="1.6.1"
ARG AUTOCONF_VERSION="2.68"
ARG LIBTOOL_VERSION="2.4.6"
ARG LOG4CXX_VERSION="0.11.0"
ARG GTEST_VERSION="1.10.0"
ARG AUTOMAKE_VERSION="1.16.3"
ARG PERL_VERSION="5.20.2"
ARG PERL_CPANVERSION="5.0"
ARG PCRE_VERSION="8.44"
ARG NGINX_VERSION="1.27.3"

# create default non-root user
#RUN groupadd -r swuser && useradd -u 1000 -g swuser -m -s /sbin/nologin -c "default non-root user" swuser

RUN chmod -R 777 /opt

RUN dnf update -y && \
    dnf install -y \
        wget \
        zlib-devel \
        openssl-devel \
        java-17-amazon-corretto \
        java-17-amazon-corretto-devel \
        pcre-devel \
        xz \
        httpd && \
    dnf clean all

# install python 2.7.8
RUN dnf update -y && \
    dnf install -y \
        python3 \
        python3-devel \
        python3-pip && \
    dnf clean all


# install git required for grpc
RUN dnf install git -y \
  && dnf clean all

RUN dnf install -y wget && dnf clean all
# Install required dependencies
RUN dnf install -y wget tar && dnf clean all

# install cmake 3.20
RUN wget https://cmake.org/files/v${CMAKE_VERSION}/cmake-${CMAKE_0VERSION}-linux-x86_64.tar.gz \
  && tar -xvf cmake-${CMAKE_0VERSION}-linux-x86_64.tar.gz \
  && cd cmake-${CMAKE_0VERSION}-linux-x86_64 \
  && cp -rf * /usr/local/ \
  && cd .. && rm -rf cmake-${CMAKE_0VERSION}-linux-x86_64.tar.gz

# install grpc. If planning to upgrade, make sure sed command works

RUN dnf install -y \
        git \
        gcc \
        gcc-c++ \
        make \
        cmake \
        automake \
        autoconf \
        protobuf-devel \
        grpc-devel \
        libtool \
        openssl-devel \
        zlib-devel \
        unzip \
    && dnf clean all
    

RUN git clone https://github.com/grpc/grpc \
  && cd grpc \
  && git checkout tags/v${GRPC_VERSION} -b v${GRPC_VERSION} \
  && git submodule update --init \
  && sed -i "s/target_link_libraries(bssl ssl crypto)/target_link_libraries(bssl ssl crypto rt)/g" third_party/boringssl-with-bazel/CMakeLists.txt \
  && mkdir -p cmake/build \
  && cd cmake/build \
  && cmake ../.. -DgRPC_INSTALL=ON -DCMAKE_BUILD_TYPE=Release -DgRPC_ABSL_PROVIDER=module -DgRPC_CARES_PROVIDER=module -DgRPC_PROTOBUF_PROVIDER=module -DgRPC_RE2_PROVIDER=module -DgRPC_SSL_PROVIDER=module -DgRPC_ZLIB_PROVIDER=module \
  && make \
  && make install \
  && cd ../../..

# Create dependencies folder to store all dependencies.
RUN mkdir -p dependencies
RUN dnf install -y wget && dnf clean all
# install boost version 1.75.0
#RUN wget https://boostorg.jfrog.io/artifactory/main/release/${BOOST_VERSION}/source/${BOOST_FILENAME}_rc1.tar.gz \
RUN wget https://archives.boost.io/release/${BOOST_VERSION}/source/${BOOST_FILENAME}.tar.gz \
    # && tar -xvf ${BOOST_FILENAME}_rc1.tar.gz \
    && tar -xvf ${BOOST_FILENAME}.tar.gz \
    && cd ${BOOST_FILENAME} \
    && ./bootstrap.sh --with-libraries=filesystem,system --prefix=/dependencies/boost/${BOOST_VERSION}/ \
    && ./b2 install define=BOOST_ERROR_CODE_HEADER_ONLY link=static threading=multi cxxflags="-fvisibility=hidden -fPIC" cflags="-fvisibility=hidden -fPIC" \
    # && cd .. && rm -rf ${BOOST_FILENAME} && rm ${BOOST_FILENAME}_rc1.tar.gz
    && cd .. && rm -rf ${BOOST_FILENAME} && rm ${BOOST_FILENAME}.tar.gz

# install opentelemetry
RUN mkdir -p dependencies/opentelemetry/${OPENTELEMETRY_VERSION}/lib \
    && mkdir -p dependencies/opentelemetry/${OPENTELEMETRY_VERSION}/include \
    && git clone https://github.com/open-telemetry/opentelemetry-cpp \
    && cd opentelemetry-cpp/ \
    && git checkout tags/v${OPENTELEMETRY_VERSION} -b v${OPENTELEMETRY_VERSION} \
    && git submodule update --init --recursive \
    && mkdir build \
    && cd build \
    && cmake .. -DCMAKE_POSITION_INDEPENDENT_CODE=ON \
                 -DBUILD_TESTING=OFF \
                 -DBUILD_SHARED_LIBS=OFF \  
                 -DWITH_OTLP=ON \
                 -DCMAKE_INSTALL_PREFIX=/dependencies/opentelemetry/${OPENTELEMETRY_VERSION} \
    && cmake --build . --target all \
    && cd .. \
    && find . -name "*.a" -type f -exec cp {} /dependencies/opentelemetry/${OPENTELEMETRY_VERSION}/lib/ \; \
    && find . -name "*.so" -type f -exec cp {} /dependencies/opentelemetry/${OPENTELEMETRY_VERSION}/lib/ \; \
    && cp -r api/include/ /dependencies/opentelemetry/${OPENTELEMETRY_VERSION}/ \
    && for dir in exporters/*; do if [ -d "$dir" ]; then cp -rf "$dir/include" /dependencies/opentelemetry/${OPENTELEMETRY_VERSION}/; fi; done \
    && cp -r sdk/include/ /dependencies/opentelemetry/${OPENTELEMETRY_VERSION}/ \
    && cp -r build/generated/third_party/opentelemetry-proto/opentelemetry/proto/ /dependencies/opentelemetry/${OPENTELEMETRY_VERSION}/include/opentelemetry/ \
    && cd .. && rm -rf opentelemetry-cpp


# install nginx

RUN dnf install -y wget && dnf clean all
RUN wget http://nginx.org/download/nginx-${NGINX_VERSION}.tar.gz \
    && tar -xvf nginx-${NGINX_VERSION}.tar.gz -C /build-dependencies \
    && rm -rf nginx-${NGINX_VERSION}.tar.gz

# Build Webserver Module
COPY . /otel-webserver-module

RUN cp -r /dependencies /otel-webserver-module/ \
    && cp -r /build-dependencies /otel-webserver-module/ \
    && cd otel-webserver-module \
    && ./gradlew assembleWebServerModule 

RUN echo '[nginx]' >> /etc/dnf.repos.d/nginx.repo \
    && echo 'name=nginx repo' >> /etc/dnf.repos.d/nginx.repo \
    && echo 'baseurl=https://nginx.org/packages/mainline/amzn/2023/x86_64/' >> /etc/dnf.repos.d/nginx.repo \
    && echo 'gpgcheck=0' >> /etc/dnf.repos.d/nginx.repo \ 
    && echo 'enabled=1' >> /etc/dnf.repos.d/nginx.repo \
    && dnf install nginx-${NGINX_VERSION} -y

RUN cp /otel-webserver-module/conf/nginx/opentelemetry_module.conf /opt/ \
    && sed -i '8i load_module /opt/opentelemetry-webserver-sdk/WebServerModule/Nginx/1.27.3/ngx_http_opentelemetry_module.so;' /etc/nginx/nginx.conf \
    && sed -i '33i include /opt/opentelemetry_module.conf;' /etc/nginx/nginx.conf \ 
    && cd /


COPY entrypoint.sh /usr/local/bin/
ENTRYPOINT ["entrypoint.sh"]