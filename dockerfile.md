# Dockerfile

1. basic image: e.g. `FROM nginx:latest` or `FROM nginx`.

  - `FROM scratch`: build from an empty image.

2. run shell command: e.g. `RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html`

  - Note: each time you use `RUN`, you add a new overlay to the basic image. This will make image too large and make the build slower. So, you should always see the following in a Dockerfile:
  
  ```
  FROM debian:stretch

  RUN buildDeps='gcc libc6-dev make wget' \
      && apt-get update \
      && apt-get install -y $buildDeps \
      && wget -O redis.tar.gz "http://download.redis.io/releases/redis-5.0.3.tar.gz" \
      && mkdir -p /usr/src/redis \
      && tar -xzf redis.tar.gz -C /usr/src/redis --strip-components=1 \
      && make -C /usr/src/redis \
      && make -C /usr/src/redis install \
      && rm -rf /var/lib/apt/lists/* \
      && rm redis.tar.gz \
      && rm -r /usr/src/redis \
      && apt-get purge -y --auto-remove $buildDeps
  ```
  
  - Pay attention to the last few lines: we clear the cache and remove build tools, in order to make the image smaller.
