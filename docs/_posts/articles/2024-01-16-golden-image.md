---
layout: post
title:  "Golden Image"
categories: posts
subtitle: Golden Image
accent_image: /assets/img/blog/golden-image/side.png
accent_color: '#ffd966'
image:
  path:    /assets/img/blog/golden-image/banner.png
  srcset:
    1920w: /assets/img/blog/golden-image/banner.png
---
* this unordered seed list will be replaced by the toc
{:toc .large-only}

## Introduction

I changed my laptop and made a decision to avoid repeating the same mistakes. My previous laptop was cluttered with unused packages and numerous dependencies, which made updating it a time-consuming task.

While the new hardware certainly improves performance, it's also refreshing to have a clean machine, and I want to maintain it that way.

First, I considered my wants and needs, and here are the main points I have in mind:

- **Install only the essentials on the laptop**: The fewer installations we have, the fewer dependencies we'll encounter. Therefore, we aim to utilize containers as much as possible.
- **Stay updated**: We want to benefit from security and feature updates. If we encounter any issues with a tool, we want to be confident that we are using the latest version, which can save time with support.
- **Minimize additional work**: This project is intended to save us time, not add more work! Thus, we want to rely on bots that will handle the updates for us, effortlessly preparing the latest version of our image.

I decided to utilize a `Docker` image where I can install all the necessary tools. This approach also offers the advantage of portability, allowing me to reuse the same image anywhere (assuming there is a Docker engine available) and instantly have my complete setup.

This article shares my experience.

Some of the tools I use are quite critical like secret manager etc… This is the reason why I keep my repository private and will share only samples here.
{:.note title="💭 Wisdom"}

## The Dockerfile

We deviate a bit from the Docker philosophy by installing multiple binaries and services in the same image, instead of having one image dedicated to a single function. Whenever feasible, we will attempt to use aliases to launch another container, but for many tools, this approach is not viable.

### Structure

Since we have a lot to install, we'll need to optimize the image. Here is the overall structure of the Dockerfile.

![placeholder](/assets/img/blog/golden-image/schema.png){:style="display:block; margin-left:auto; margin-right:auto"}

During the build, the Builder will be created first, and then the different builders will run in parallel to optimize the build time.
This approach also makes it easier to read and modify when necessary.

### Builder

```dockerfile
# Create Builder
FROM debian:bookworm@sha256:b16cef8cbcb20935c0f052e37fc3d38dc92bfec0bcfb894c328547f81e932d67 as builder

# Github-releases
ENV ACT_VER v0.2.57
ENV STARSHIP_VER v1.17.1
...SNIPPED...

# Repology
# renovate: datasource=repology depName=debian_12/bat versioning=loose
ENV BAT_VER 0.22.1
...SNIPPED...

# Register arch names
ARG TARGETPLATFORM
SHELL ["/bin/bash", "-o", "pipefail", "-c"]
RUN dpkgArch="$(echo "$TARGETPLATFORM" | cut -d '/' -f2)" && \
    case "${dpkgArch##*-}" in \
    'amd64') echo 'starship-x86_64-unknown-linux-gnu' > /tmp/ARCH_STARSHIP; echo 'amd64' > /tmp/ARCH_DOCKER; echo 'x86_64' > /tmp/ARCH_LAZYDOCKER; echo 'x86_64' > /tmp/ARCH_DOCKERPLUGIN ;; \
    'arm64') echo 'starship-aarch64-unknown-linux-musl' > /tmp/ARCH_STARSHIP; echo 'arm64' > /tmp/ARCH_DOCKER; echo 'arm64' > /tmp/ARCH_LAZYDOCKER; echo 'aarch64' > /tmp/ARCH_DOCKERPLUGIN ;; \
    *) echo "unsupported architecture"; exit 1 ;; \
    esac
SHELL ["/bin/sh", "-c"]

# Install base packages
# False positive in HADOLINT, the version is pinned
# hadolint ignore=DL3008
RUN apt-get update && apt-get install --no-install-recommends -y \
    "build-essential=${BUILDESSENTIAL_VER}*" \
    ca-certificates=${CACERTIFICATES_VER} \
    ...SNIPPED...
    zsh=${ZSH_VER} && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

The builder stores the **versions** of the packages we install in environment variables, which are modified by `renovate` (see below).

The next step is to identify the **architecture** for building (using a building argument) and generate files with different syntax to hold the architecture name. These files will be used by the application builders to download the appropriate package.

Lastly, we install certain **dependencies** that are required by the application builders and are also needed in the final image.

### Application Builders

```dockerfile
# Act Builder
FROM builder as act-builder
SHELL ["/bin/bash", "-o", "pipefail", "-c"]
RUN export ARCH_ACT=$(cat "/tmp/ARCH_LAZYDOCKER") && \
    wget -q https://github.com/nektos/act/releases/download/${ACT_VER}/checksums.txt -O /tmp/act_checksums.txt && \
    wget -q https://github.com/nektos/act/releases/download/${ACT_VER}/act_Linux_${ARCH_ACT}.tar.gz -O /tmp/act_Linux_${ARCH_ACT}.tar.gz && \
    grep "act_Linux_${ARCH_ACT}.tar.gz" /tmp/act_checksums.txt | sed "s|act_Linux_${ARCH_ACT}.tar.gz|/tmp/act_Linux_${ARCH_ACT}.tar.gz|" | shasum -c - && \
    tar -xf "/tmp/act_Linux_${ARCH_ACT}.tar.gz" -C /tmp/ && \
    chmod +x "/tmp/act" && \
    mv "/tmp/act" "/usr/bin/"
SHELL ["/bin/sh", "-c"]

...SNIPPED...

# Dos2unix Builder
FROM builder as dos2unix-builder
RUN apt-get update && apt-get install --no-install-recommends -y \
    "dos2unix=${DOS2UNIX_VER}*" && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

...SNIPPED...

# TheFuck Builder
FROM builder as thefuck-builder
RUN python3 -m venv /thefuck-env && \
    /thefuck-env/bin/pip install --no-cache-dir "thefuck==${THEFUCK_VER}" && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

I may need to improve the consistency of this section…
{:.note title="💭 Wisdom"}

One difficulty I encountered is with packages that have a version containing an epoch and a debian-revision. 
The debian-revision can be addressed with a wildcard ${PACKAGE_VERSION}*, but the epoch is a bit trickier. 
For those packages, you need to use a search pattern:

```bash
"?and(?exact-name(<PACKAGE_NAME>),?version(:${PACKAGE_VERSION}))"
```

### Final layer

```dockerfile
# Final Image
FROM builder
ARG USER=jerome

LABEL org.opencontainers.image.description "Jerome Ducret's personal Docker image containing usefull tools. See https://github.com/DucretJe/base-image/"

RUN addgroup --system ${USER} && \
    adduser --home /home/${USER} --system --ingroup sudo --ingroup ${USER} ${USER} && \
    echo "${USER} ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers

# Install Act
COPY --from=act-builder /usr/bin/act /usr/bin/act

...SNIPPED...

HEALTHCHECK NONE
WORKDIR /home/${USER}

SHELL ["/usr/bin/zsh"]
ENTRYPOINT [ "/usr/bin/zsh" ]
CMD [ ]
```
Here, I declare the user I will use most of the time and retrieve the binaries I need. That's pretty much it.

Note that when I can, I don’t install the binary but instead I create an alias to run the dockerized version of the application. I copy a file with all my aliases and import it in my .zshrc
{:.note title="💭 Wisdom"}

## Version Update

### Renovate

The idea is to automate the update process completely, eliminating the need to worry about it.

Spoiler alert: It's not magical. Sometimes, updates can get stuck due to changes in versioning or failed runners. However, most of them are automatically managed.
{:.note title="💭 Wisdom"}

I choose to use "renovate" because it is easy to customize according to our needs.

As mentioned earlier, all the versions are declared as variables in the first layer of our `dockerfile`, so we simply need to specify it to renovate. I use two different methods:

- **The GitHub releases**: It's the simplest option, but it needs to be declared for each package in the renovate configuration. Additionally, it's not always available.
```json
"regexManagers": [
    {
      "fileMatch": [
        "^docker/.*Dockerfile$"
      ],
      "matchStrings": [
        "ACT_VER (?<currentValue>.*)"
      ],
      "depNameTemplate": "nektos/act",
      "datasourceTemplate": "github-releases"
    },
```
- The version of the package manager: It's a bit tricky, but since we are using a regex, all versions are detected. The regex captures the comment I added before each environment variable to indicate where to look and what to look for.
```json
{
      "fileMatch": [
        "^docker/.*Dockerfile$"
      ],
      "matchStrings": [
        "#*renovate: datasource=(?<datasource>.*?) depName=(?<depName>.*?) versioning=(?<versioning>.*?)\nENV .*_VER (?<currentValue>.*?)\n"
      ],
      "versioningTemplate": "{{#if versioning}}{{versioning}}{{else}}semver{{/if}}"
}
```

One improvement would be to reuse the logic of Repology for the Github Releases feature. I didn't do it because I was afraid of breaking something, but it would be a good idea to do it.
{:.note title="💭 Wisdom"}

I don't want to spend all my time approving pull requests generated by Renovate, which is why I allow it to auto-merge them. However, to maintain some control, I had to add some mandatory tests.

### Tests

I wanted to keep my tests simple, so I use a makefile to:

- Build the image using `buildx` for multi-architecture support.
- Run the created image and modify the `entrypoint` to print the version of each application. This confirms that the applications are present in the image and can start successfully.

Additionally, I want to test all the apps even if one test fails, so that I can identify the specific issue with the build if necessary.

For this purpose, I created a test function:

```makefile
define test_command
	@echo ${ORANGE}Testing $(1) 🔬${LGRAY} && \
	docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v `pwd`:/rec --pull always --entrypoint $(1) $(IMAGE_NAME) $(2); \
	if [ $$? = 0 ]; then \
		echo ${GREEN}$(1) OK ✅${NC}; \
	else \
		echo ${RED}$(1) test failed 🛑${NC}; \
		echo "1" > test-fail; \
	fi
endef
```

I also created a build step:

```makefile
build:
	@cd ../docker && \
	docker buildx create --name multiarchbuilder --use && \
	docker buildx build --platform linux/amd64,linux/arm64 -t $(IMAGE_NAME) .
	docker buildx rm multiarchbuilder
```

And multiple checks like:

```makefile
act:
	$(call test_command,act,--version)
```

Finally a final check that will return a `rc` 1 if one of the tests failed.

```makefile
check:
	@if [ -f test-fail ]; then \
		rm test-fail; \
		echo ${RED} Some tests failed; \
		exit 1; \
	fi
```

## Caveheats

In general, I'm happy with my image. I love starting the day with a new one and seeing all the packages up to date. I also appreciate the quick system upgrades because I have a low number of directly installed packages.

Using the same environment across all the machines I work on is beneficial for maintaining productivity.

However, it's not 100% perfect. There are some caveats 😔.

### Mounting volumes

As mentioned earlier, I often use aliases to run dockerized versions of certain applications. Some of these applications require mounting a directory, such as [superlinter](https://github.com/super-linter/super-linter). However, when we are already inside a container, using a mounted directory would require us to specify the path from the host filesystem, rather than the current container.

While it is generally recommended to avoid complex mount points, in this specific use case, we need to use them.

When I use this image, I mount my working directory that contains all the git repositories I am working with. Therefore, I am usually in the working directory or a subdirectory of it. This means that it is possible to reconstruct the host's path.

To achieve this, I use two environment variables that I define with my `docker run` command:

- `HOST_PWD=$(pwd)`
- `WORKDIR=workdir`

Workdir is the same value than the --workdir flag of my docker run command.
{:.note title="💭 Wisdom"}

Finally, I declare an alias to use instead of `pwd` in my container.

```bash
alias hostpwd='echo "$HOST_PWD$(echo `pwd` | sed 's/$WORKDIR//')"'
```

### Clipboard

I use `vim` as my IDE within my container. It allows me to conveniently work from my terminal without needing to switch to another window when working on my code. It also eliminates the need to install an IDE on the host system.

While `vim` is great, there is a small issue. We can use `xclip` to share the clipboard of `vim` with the host, but the problem is that our container does not have an X11 instance, making it impossible to work.

One solution is to install `xQuartz` on the host and mount the `x11` socket in the container. However, this introduces another dependency on the host system, which I am not enthusiastic about. I tried this approach and discovered that it is indeed possible to link the container's clipboard to the host's clipboard. However, there is a bug preventing it from being refreshed: I had to modify the options to retrieve the content of the container's clipboard on the host's clipboard, and if I add something else, I have to modify the options again to see it on my host's clipboard.

Unfortunately, I have not yet found a viable solution.

### Multi architecture & Build time

Since I can work on both ARM and x86 CPUs, I want to build images for both architectures. While it is possible to build in x86 and use emulation to run it on ARM systems, I wanted to try building for both architectures separately.

I use the Github runners to build my images, but I encountered an issue. Due to the size of my Dockerfile, it takes around 20 minutes to build it with a small runner. However, for an ARM runner, it takes three times as long. Since I build the image for every PR to test it, and I have many dependencies (resulting in frequent update PRs), it consumes all my Github action minutes. Furthermore, the repository is private.

To address this, I had to create private runners, one for x86 and one for ARM. This allows me to build my images multiple times a day without exhausting my action minutes.

## Wrap Up

I have been working on this image for approximately 6 months, consistently trying to improve it and fix any issues that arise. It's a work in progress that never really ends, but overall, I am quite satisfied with it.

On average, I unblock about 1 pull request per month, and I frequently add new apps to the image.

Now, it is starting to reach a mature state, and the user experience is quite good. Throughout this project, I have gained a lot of knowledge about Dockerfiles, makefiles for testing, and building for multiple architectures. Because of these reasons, I consider this project to be a success. 🙂
