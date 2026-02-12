---
title: "Docker Image Layering"
subtitle: "Building custom Docker images from running containers and understanding how layers work."
date: 2022-02-24
tags:
  - docker
  - images
  - getting-started
categories:
  - Docker Fundamentals
readtime: true
---
Building custom Docker images from running containers and understanding how layers work.
<!--more-->

This continues from [Docker Images Basics](/post/docker-images-basics/).

> *"Every layer is permanent. Once committed, it's in the history forever. Plan your layers."* - Docker image hygiene

![Docker Slander - It also "works on my machine"](/assets/img/memes/docker-slander.png)

---

## Pull and Run a Base Image

Start with a pinned version of `httpd`:

```bash
docker pull httpd:2.4
```

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

Pin image versions (`httpd:2.4`) instead of using `:latest`. Pinning ensures you get the same
image every time, which matters when debugging or rolling back.

</div>
</div>

Run a container from the image:

```bash
docker run --name web_image -d httpd:2.4
```

Note the base image size for comparison later:

```bash
docker images

REPOSITORY   TAG       IMAGE ID       CREATED       SIZE
httpd        2.4       a8ea074f4566   4 weeks ago   144MB
```

---

## Modify the Running Container

Exec into the container:

```bash
docker exec -it web_image bash
```

Install `git`, clone web content, and replace the default `htdocs`:

```bash
apt update && apt install git -y

git clone https://github.com/YOUR-USERNAME/html5up-solid-state.git /tmp/html5up-solid-state

rm htdocs/index.html
cp -r /tmp/html5up-solid-state/* htdocs/

exit
```

---

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

Before committing, run `docker diff web_image` to see exactly what changed in the container's
filesystem. `A` = added, `C` = changed, `D` = deleted. This is useful for verifying you only
modified what you intended before baking it into an image.

</div>
</div>

## Commit the Container as a New Image

Save the running container state as `v1`:

```bash
docker commit $(docker ps -q -f name=web_image) example/web_image:v1
```

Check the image sizes:

```bash
docker images

REPOSITORY          TAG       IMAGE ID       CREATED          SIZE
example/web_image   v1        ad1a7ce03f5b   2 minutes ago   251MB
httpd               2.4       a8ea074f4566   4 weeks ago     144MB
```

The v1 image is 251 MB because it includes `git`, its dependencies, and the cloned repo in `/tmp`.

I once shipped a 2 GB "minimal" image to production. Forgot to remove the build tools and a 1.5 GB cache directory before committing. The deployment worked. The pull times did not. A colleague asked why our CI was suddenly 8 minutes slower. Layers don't lie. `docker history` told the whole story.

---

## Clean Up and Create v2

Exec back in and remove the build artifacts:

```bash
docker exec -it web_image bash

rm -rf /tmp/html5up-solid-state/
apt remove git -y && apt autoremove -y && apt clean

exit
```

Commit the cleaned-up state as `v2`:

```bash
docker commit $(docker ps -q -f name=web_image) example/web_image:v2
```

Compare sizes:

```bash
docker images

REPOSITORY          TAG       IMAGE ID       CREATED          SIZE
example/web_image   v2        0ae12ea5ded7   38 seconds ago   168MB
example/web_image   v1        ad1a7ce03f5b   14 minutes ago   251MB
httpd               2.4       a8ea074f4566   4 weeks ago      144MB
```

v2 is 168 MB - closer to the base image because the build tools were removed.

<div class="admonition admonition-warning" markdown="1">
<div class="admonition-title">⚠️ Warning</div>
<div class="admonition-content" markdown="1">

Even though v2 is smaller than v1, the removed files still exist in the v1 layer. Docker layers
are additive - removing files in a later commit adds a new layer that *hides* the old files but
doesn't reclaim the space from the base layer.

> *"Layers are append-only. Deletes don't shrink - they mask. Multi-stage builds exist for a reason."* - Image optimization wisdom

*"There is no spoon."* - The Matrix. There is no delete. There's only another layer on top. This is why Dockerfiles with multi-stage builds produce smaller images than `docker commit` workflows.

</div>
</div>

Delete v1:

```bash
docker rmi example/web_image:v1
```

---

## Run Multiple Containers from the Image

```bash
docker run -d --name web1 -p 8081:80 example/web_image:v2
docker run -d --name web2 -p 8082:80 example/web_image:v2
docker run -d --name web3 -p 8083:80 example/web_image:v2
```

Verify:

```bash
docker ps

CONTAINER ID   IMAGE                  COMMAND              CREATED          STATUS          PORTS                                   NAMES
6c1ede5cb1f9   example/web_image:v2   "httpd-foreground"   47 seconds ago   Up 46 seconds   0.0.0.0:8083->80/tcp, :::8083->80/tcp   web3
10082d295276   example/web_image:v2   "httpd-foreground"   47 seconds ago   Up 46 seconds   0.0.0.0:8082->80/tcp, :::8082->80/tcp   web2
2c0394bac748   example/web_image:v2   "httpd-foreground"   48 seconds ago   Up 47 seconds   0.0.0.0:8081->80/tcp, :::8081->80/tcp   web1
```

Quick health check:

```bash
curl -I localhost:8081 | grep HTTP
HTTP/1.1 200 OK

curl -I localhost:8082 | grep HTTP
HTTP/1.1 200 OK

curl -I localhost:8083 | grep HTTP
HTTP/1.1 200 OK
```

---

## Visualizing Layers

`docker history` shows the layer stack. Compare the base image and custom image:

```bash
docker history httpd:2.4

IMAGE          CREATED       CREATED BY                                      SIZE      COMMENT
a8ea074f4566   4 weeks ago   /bin/sh -c #(nop)  CMD ["httpd-foreground"]     0B
<missing>      4 weeks ago   /bin/sh -c #(nop)  EXPOSE 80                    0B
<missing>      4 weeks ago   /bin/sh -c set -eux;   savedAptMark="$(apt-m…   60.5MB
```

```bash
docker history example/web_image:v2

IMAGE          CREATED             CREATED BY                                      SIZE      COMMENT
0ae12ea5ded7   About an hour ago   httpd-foreground                                24.3MB
a8ea074f4566   4 weeks ago         /bin/sh -c #(nop)  CMD ["httpd-foreground"]     0B
<missing>      4 weeks ago         /bin/sh -c #(nop)  EXPOSE 80                    0B
<missing>      4 weeks ago         /bin/sh -c set -eux;   savedAptMark="$(apt-m…   60.5MB
```

The custom image adds a single 24.3 MB layer on top of the base `httpd:2.4` layers. Each
`docker commit` creates one new layer containing only the filesystem diff.

---

## Use Dockerfiles Instead

The `docker commit` workflow above is useful for understanding layers. For anything real, use
a **Dockerfile**:

```dockerfile
FROM httpd:2.4
COPY ./my-site/ /usr/local/apache2/htdocs/
```

One file, same result, version-controlled.

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

See the [Dockerfile reference](https://docs.docker.com/reference/dockerfile/) for the full
syntax. Multi-stage builds can further reduce image size by separating build tools from the
final image.

</div>
</div>

---

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

Run `docker system prune -a` periodically to remove unused images, stopped containers, and
dangling build cache. Add `--volumes` to also clean up unused volumes. Check what would be
removed first with `docker system df`.

</div>
</div>

---

## References

- [Docker Images Basics](/post/docker-images-basics/) - Prerequisite: pull, run, volumes
- [Getting Started with Docker](/post/docker-getting-started/) - Installation
- [Docker CLI reference](https://docs.docker.com/reference/cli/docker/)
- [Dockerfile reference](https://docs.docker.com/reference/dockerfile/)

---

## Acknowledgements

Influenced by Docker training originally available at A Cloud Guru
(now part of [Pluralsight](https://www.pluralsight.com/cloud-guru)).
