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

### Dockerfile commands

1. COPY

  - examples: `COPY package.json /usr/src/app/`, `COPY hom* /mydir/`, `COPY package.json mydir/`(relative path to `WORKDIR`).
  - No need to create directories for non-existing paths.
  - Support `--chown=<user>:<group>` to change user/group.

2. ADD

  - It has similar usage to `COPY`, But it can download and auto unzip the file to the target directory.
  - So, `COPY` is recommended for copying files.

3. CMD
 
  - `CMD` specifies the default command of the container, e.g. `ubuntu`'s is `/bin/bash`. And we can overwrite it at start, e.g. `docker run -it ubuntu cat /etc/os-release`.
  - shell form: `CMD <command>`. e.g. `CMD echo $HOME`, will be translated to `CMD [ "sh", "-c", "echo $HOME" ]`.
  - exec form(recommended): `CMD [ "<exe-file>", "arg1", "arg2" ]` e.g. `CMD [ "sh", "-c", "echo $HOME" ]`.
  - Note: container is a **process**, so `CMD service nginx start` will make the container exit immediately because it's translated to `CMD [ "sh", "-c", "service nginx start"]`.

4. ENTRYPOINT

  - It has same usage as `CMD`, supporting shell form and exe form.
  - You can also replace the default entrypoint by `--entrypoint`.
  - You can specify **additional** arguments to the entrypoint. e.g. `ENTRYPOINT [ "curl", "-s", "http://ip.cn" ]`, you can use `docker run <image> -i` to get only the headers. Otherwise, if you are using `CMD`, you must use `docker run <image> curl -s "http://ip.cn" -i` to do the same thing.
  - Another usage is to configure the image according to the arguments we use. Take redis image as example, it will run `docker-entrypoint.sh redis-server` by default. If we run with specific arguments, it will run `docker-entrypoint.sh <user's-arguments>`.
  
  ```dockerfile
  FROM alpine:3.4
  ...
  RUN addgroup -S redis && adduser -S -G redis redis
  ...
  ENTRYPOINT ["docker-entrypoint.sh"]

  EXPOSE 6379
  CMD [ "redis-server" ]
  ```

5. ENV

  - `ENV <key> <value>` or `ENV <key1>=<value1> <key2>=<value2>...`.
  - Support `ADD`、`COPY`, `ENV`, `EXPOSE`, `FROM`, `LABEL`, `USER`, `WORKDIR`, `VOLUME`, `ONBUILD`, `RUN`.
  - We can also specify the env at running the container, e.g. `docker run -e REDIS_NAMESPACE='staging'`.

6. ARG

  - `ARG <key>=<value>`
  - It's used at `docker build` and will **not** be seen during runtime.
  - We can use `docker build --build-arg <key>=<value>` to overwrite it.
 
7. VOLUME
 
  - `VOLUME <path>` or `VOLUME ["<path1>", "path2", ...]`.
  - It's the default (**anonymous**) volume, in case we forget to mount a volume at runtime.
  - e.g. we can overwrite it using `-v mydata:/data` for `VOLUME /data`. 

8. WORKDIR

  - `WORKDIR <path>`.
  - We can use it to set current working directory for the following commnads in `Dockerfile` during build. And there's no need to create non-existing directories.

9. USER

  - `USER <user>[:<group>]`.
  - Swith user for the following commands. But we need to create the user before switching.

10. HEALTHCHECK

  - Help Docker cli to understand container's health status: https://vuepress.mirror.docker-practice.com/image/dockerfile/healthcheck.html

11. ONBUILD

  - Execute some commands when this image serves as the basic image during another image build: https://vuepress.mirror.docker-practice.com/image/dockerfile/onbuild.html


12. EXPOSE

  - Just a declaration: https://vuepress.mirror.docker-practice.com/image/dockerfile/expose.html

### Multi-stage builds

If we want to use a single `Dockerfile`, we will have the following downsides.

- introduce too many overlays because of compiling, testing, packing, etc.
- cannot protect source code.

Then, we might want to use two `Dockerfile`s, one for compiling/testing and one for deployment. It works, but not very convenient.
So, the ultimate solution is `multi-stage builds`, building multiple images use a single `Dockerfile`: https://vuepress.mirror.docker-practice.com/image/multistage-builds/


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
