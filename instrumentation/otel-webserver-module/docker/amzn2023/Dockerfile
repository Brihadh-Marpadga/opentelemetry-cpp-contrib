FROM amazonlinux:2023

ARG NGINX_VERSION="1.27.3"

# install nginx
RUN dnf install -y wget gzip tar && dnf clean all
RUN mkdir -p /build-dependencies && \
    wget http://nginx.org/download/nginx-${NGINX_VERSION}.tar.gz \
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