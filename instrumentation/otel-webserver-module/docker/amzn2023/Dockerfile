FROM amazonlinux:2023

ARG BUILD_NUMBER

LABEL NAME=apm/build-image-webserver-agent-centos6-x64 VERSION=$BUILD_NUMBER

ENV GOSU_ARCH amd64
ENV JDK_ARCH x64

ARG PYTHON_VERSION="3.9.5"
ARG GRPC_VERSION="1.60.0"
ARG OPENTELEMETRY_VERSION="1.2.0"
ARG BOOST_VERSION="1.75.0"
ARG BOOST_FILENAME="boost_1_75_0"
ARG NGINX_VERSION="1.27.3"

# Install required utilities
RUN dnf install -y shadow-utils && dnf clean all

# Create default non-root user
RUN groupadd -r swuser && useradd -u 1000 -g swuser -m -s /sbin/nologin -c "default non-root user" swuser

RUN chmod -R 777 /opt

# Install required packages via dnf
RUN dnf update -y && dnf install -y \
    gcc gcc-c++ make cmake automake autoconf libtool \
    java-17-amazon-corretto java-17-amazon-corretto-devel \
    httpd git shadow-utils \
    wget gzip \
    openssl-devel zlib-devel bzip2 xz-devel libffi-devel \
    && dnf clean all

# Install Python 3.9
RUN wget http://www.python.org/ftp/python/${PYTHON_VERSION}/Python-${PYTHON_VERSION}.tar.xz \
    && unxz Python-${PYTHON_VERSION}.tar.xz \
    && tar -xf Python-${PYTHON_VERSION}.tar \
    && cd Python-${PYTHON_VERSION} && ./configure --prefix=/usr/local && make && make install \
    && cd .. && rm -rf Python-${PYTHON_VERSION}*

# Install GOSU
ENV GOSU_VERSION=1.10
RUN curl -o /usr/bin/gosu -SL "https://github.com/tianon/gosu/releases/download/${GOSU_VERSION}/gosu-amd64" \
    && chmod +x /usr/bin/gosu \
    && gosu nobody true  # Verify installation

# Install gRPC
RUN git clone https://github.com/grpc/grpc \
  && cd grpc \
  && git checkout tags/v${GRPC_VERSION} -b v${GRPC_VERSION} \
  && git submodule update --init \
  && sed -i "s/target_link_libraries(bssl ssl crypto)/target_link_libraries(bssl ssl crypto rt)/g" third_party/boringssl-with-bazel/CMakeLists.txt \
  && mkdir -p cmake/build \
  && cd cmake/build \
  && cmake ../.. -DgRPC_INSTALL=ON -DCMAKE_BUILD_TYPE=Release -DgRPC_ABSL_PROVIDER=module -DgRPC_CARES_PROVIDER=module -DgRPC_PROTOBUF_PROVIDER=module -DgRPC_RE2_PROVIDER=module -DgRPC_SSL_PROVIDER=module -DgRPC_ZLIB_PROVIDER=module \
  && make && make install \
  && cd ../../.. && rm -rf grpc

# Create dependencies folder
RUN mkdir -p dependencies

# Install Boost
RUN wget https://archives.boost.io/release/${BOOST_VERSION}/source/${BOOST_FILENAME}.tar.gz \
    && tar -xvf ${BOOST_FILENAME}.tar.gz \
    && cd ${BOOST_FILENAME} \
    && ./bootstrap.sh --with-libraries=filesystem,system --prefix=/dependencies/boost/${BOOST_VERSION}/ \
    && ./b2 install define=BOOST_ERROR_CODE_HEADER_ONLY link=static threading=multi cxxflags="-fvisibility=hidden -fPIC" \
    && cd .. && rm -rf ${BOOST_FILENAME}*

# Install OpenTelemetry
RUN mkdir -p dependencies/opentelemetry/${OPENTELEMETRY_VERSION}/lib \
    && mkdir -p dependencies/opentelemetry/${OPENTELEMETRY_VERSION}/include \
    && git clone https://github.com/open-telemetry/opentelemetry-cpp \
    && cd opentelemetry-cpp/ \
    && git checkout tags/v${OPENTELEMETRY_VERSION} -b v${OPENTELEMETRY_VERSION} \
    && git submodule update --init --recursive \
    && mkdir build \
    && cd build \
    && cmake .. -DCMAKE_POSITION_INDEPENDENT_CODE=ON -DBUILD_TESTING=OFF -DBUILD_SHARED_LIBS=ON -DWITH_OTLP=ON -DCMAKE_INSTALL_PREFIX=/dependencies/opentelemetry/${OPENTELEMETRY_VERSION} \
    && cmake --build . --target all \
    && cd .. \
    && find . -name "*.so" -type f -exec cp {} /dependencies/opentelemetry/${OPENTELEMETRY_VERSION}/lib/ \; \
    && cp build/libopentelemetry_proto.a /dependencies/opentelemetry/${OPENTELEMETRY_VERSION}/lib \
    && cp -r api/include/ /dependencies/opentelemetry/${OPENTELEMETRY_VERSION}/ \
    && for dir in exporters/*; do if [ -d "$dir" ]; then cp -rf "$dir/include" /dependencies/opentelemetry/${OPENTELEMETRY_VERSION}/; fi; done \
    && cp -r sdk/include/ /dependencies/opentelemetry/${OPENTELEMETRY_VERSION}/ \
    && cp -r build/generated/third_party/opentelemetry-proto/opentelemetry/proto/ /dependencies/opentelemetry/${OPENTELEMETRY_VERSION}/include/opentelemetry/ \
    && cd .. && rm -rf opentelemetry-cpp

# Install Nginx
RUN wget http://nginx.org/download/nginx-${NGINX_VERSION}.tar.gz \
    && tar -xvf nginx-${NGINX_VERSION}.tar.gz -C /build-dependencies \
    && rm -rf nginx-${NGINX_VERSION}.tar.gz

# Build Webserver Module
COPY . /otel-webserver-module

RUN cp -r /dependencies /otel-webserver-module/ \
    && cp -r /build-dependencies /otel-webserver-module/ \
    && cd otel-webserver-module \
    && ./gradlew assembleWebServerModule 

RUN echo '[nginx-stable]' >> /etc/yum.repos.d/nginx.repo \
    && echo 'name=nginx stable repo' >> /etc/yum.repos.d/nginx.repo \
    && echo 'baseurl=http://nginx.org/packages/centos/8/$basearch/' >> /etc/yum.repos.d/nginx.repo \
    && echo 'gpgcheck=1' >> /etc/yum.repos.d/nginx.repo \
    && echo 'enabled=1' >> /etc/yum.repos.d/nginx.repo \
    && echo 'gpgkey=https://nginx.org/keys/nginx_signing.key' >> /etc/yum.repos.d/nginx.repo \
    && echo 'module_hotfixes=true' >> /etc/yum.repos.d/nginx.repo \
    && dnf install nginx-${NGINX_VERSION} -y

RUN cp /otel-webserver-module/conf/nginx/opentelemetry_module.conf /opt/ \
    && sed -i '8i load_module /opt/opentelemetry-webserver-sdk/WebServerModule/Nginx/1.26.2/ngx_http_opentelemetry_module.so;' /etc/nginx/nginx.conf \
    && sed -i '33i include /opt/opentelemetry_module.conf;' /etc/nginx/nginx.conf \
    && cd /

COPY entrypoint.sh /usr/local/bin/
ENTRYPOINT ["entrypoint.sh"]
