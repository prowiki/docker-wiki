# Cheatsheet

## Common

1. run an image in a container:

```bash
docker run -it --rm busybox:latest bash
```

  - `-i`: interactive;
  - `-t`: in terminal;
  - `--rm`: remove image after exiting container.

2. list downloaded images

```bash
docker image ls
```

3. remove images

```bash
docker image rm <image-name/image-id> # image-name is <repo:tag>
```

## Image

### list

1. list dangling images: `docker image ls -f dangling=true`; delete dangling images: `docker image prune`. (`-f`: filter)
2. list by `name/tag`:  `docker image ls ubuntu` or `docker image ls ubuntu:18.04`.
3. list with customized output: `docker image ls --format "{{.ID}}: {{.Repository}}"`. (using go template)

### remove

1. delete all images with the repo name of `redis`: `docker image rm $(docker image ls -q redis)`
