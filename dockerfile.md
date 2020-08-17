# Dockerfile

## How to write Dockerfile

1. basic image: e.g. `FROM nginx:latest` or `FROM nginx`.

  - `FROM scratch`: build from an empty image.

2. run shell command: e.g. `RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html`

  - Note: each time you use `RUN`, you add a new overlay to the basic image. This will make image too large and make the build slower. So, you should always see the following in a Dockerfile:
  
  ```dockerfile
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

## Docker build

We will use the following `Dockerfile` as example.
```dockerfile
FROM nginx
RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
```

### Explanation of build process

First, we build the image from the Dockerfile: `docker image -t nginx:v3 .`. We will see the process:

```bash
Sending build context to Docker daemon  2.048kB
Step 1/2 : FROM nginx
 ---> 4bb46517cac3
Step 2/2 : RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
 ---> Running in 095379bd399d
Removing intermediate container 095379bd399d
 ---> 5a17fa4c3cb9
Successfully built 5a17fa4c3cb9
Successfully tagged nginx:v3
```

1. Docker cli is just a client, the build process is actually running Docker engine (i.e. Docker daemon). The client communiate with ther server through `REST API` (i.e. `Docker Remote API`).
2. `.` in this command indicates not only where `Dockerfile` is but also the **context**.
  - Docker cli will pack the files under the context and upload to Docker engine while building images: `Sending build context to Docker daemon  2.048kB`.
  - And this is the basic for `ADD` and `COPY`. So, you will not be able to run something like `COPY ../package.json /app` or `COPY /opt/xxxx /app`.
3. Step 1: get the basic image: `4bb46517cac3`.
4. Step 2: run a container overlay `095379bd399d` on the basic image, execute the command, and commit the changes as a new overlay `5a17fa4c3cb9`. After that, we don't need a container overlay, so delete it after build.

### Tips for Dockerfile

1. Place the `Dockerfile` in the root directory of the project, and use `.dockerignore` to exclude files that you don't want to pass to the Docker engine.
2. Use `-f ./xxx/Dockerfile.config` to specify the `Dockerfile` you want to use during build. `./Dockerfile` is just the default.
3. You can build image from git repositories or tarballs directly.
