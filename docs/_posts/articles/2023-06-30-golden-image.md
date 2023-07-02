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

Starting a new job often means new devices! So since I’m changing, I also wanted to change my workflow a bit. I've been thinking about this for a while, but I never had the time to do it, until now!

First, I thought about what I want and need, and here are the main points that I have in mind:

- **Install the bare minimum on the laptop**: The less we install, the fewer dependencies we have. Therefore, we want to use containers as much as possible.
- **Stay up to date**: We want to benefit from security and feature updates. If we have an issue with a tool, we want to be confident that we are running the latest version, which can save time with support.
- **No additional work**: This project is supposed to save us time, not add more work! So, we want to rely on bots that will do the updates for us, magically preparing the latest version of our image.

Using containers seems to be the best option for our needs. While we could have one image per tool, it would become complicated when we want to use multiple tools together. In some cases, we may need to share files, which would require us to specify mounting points. While this is possible, it may not be the most time-efficient solution, especially during emergency situations 🔥 👨‍🚒.

Therefore, I've decided to use one big image containing all the tools that I need. The **golden image**. I'll just mount it in my `pro` directory, and I'll probably stay in the container all day.

To stay up to date without having to do anything, we could install our package in `latest`, but this would cause some difficulties:

- A new image wouldn't be built at each package update.
- Our image would be mutable, and we could have inconsistent behavior. It could be built one day and fail the next day because of an incompatible dependency between two packages.

Therefore, we want to pin our versions and have a bot update them for us.

However, there is another issue. If we want to do absolutely nothing, we must let the bot update the versions without checking on them. But this could lead to broken images 😱 and I don't want to start my day debugging my golden image!

Therefore, we need to add some tests to check if the new version works.

I won't publish an exhaustive list of the tools I work with, which is why my repository is private. However, I'll present a truncated version of my image here.
{:.note title="💭 Note"}

## Renovate

I use Renovate for all of my repositories. I have a central repository for the common configuration, and my other repositories point to it. They use their custom configuration if they have particular needs.

We have the following objectives for Renovate:

- Update the base image version.
- Update the pinned package versions in our `Dockerfile` for:
    - The package manager.
    - Manually installed packages.
- Do it without us (with some conditions)

### Bump the base image

Renovate can update the base image if it uses the `SemVer` convention. My favorite Linux distribution is Debian, which uses a `SemVer` tag. Therefore, I start with `debian:12.0`.

```json
"packageRules": [
    {
      "datasources": [
        "docker"
      ],
      "updateTypes": [
        "major",
        "minor",
        "patch"
      ]
    }
  ]
```

In essence, we are requesting that Renovate monitor our Docker images and handle all major, minor, and patch updates to cover our needs.

⚠️ The major updates may cause some issues, as certain packages may not exist in the new version. However, we will be notified and these updates should not occur frequently, making it an acceptable risk.
{:.note title="💭 Note"}

Renovate should now be able to detect our `Dockerfile` and our base image in it. We can check it in the `Dependency Dashboard`:

<p align="center">
  <img src="https://github.com/DucretJe/ducretje.github.io/assets/5384298/226db278-c9c6-419d-a550-59bcbaa844df" />
</p>

There is currently no more recent version available. Renovate will not make any change.

<p align="center">
  <img src="https://github.com/DucretJe/ducretje.github.io/assets/5384298/38a4128f-66d1-4aae-95ba-81b0d48f3e00" />
</p>

From now on, we should always use an up-to-date base image! 🍾

### Bump the manually installed packages

We will need to manually install some packages because we won't be able to pin a version using the package manager, and some packages won't be available in it at all.

For example, let's look at `starship` (no, not the SpaceX one 🚀). We will install the package from [GitHub](https://github.com/starship/starship/releases). To specify a pinned version in our `DockerFile`, we will need to use the `RegexManagers` block in our renovate configuration file.

```json
"regexManagers": [
...SNIPPED...
    {
      "fileMatch": [
        "^docker/.*Dockerfile$"
      ],
      "matchStrings": [
        "STARSHIP_VER (?<currentValue>.*)"
      ],
      "depNameTemplate": "starship/starship",
      "datasourceTemplate": "github-releases"
    },
...SNIPPED...
]
```

First we specify to Renovate that the dependency is mentioned in our DockerFile.
Then we make it search the variable `STARSHIP_VER` and capture the value.
Then we specify the name of the repository and we tell Renovate that it’s a `github-release`.

That’s it, Renovate should be now monitoring this repository and will update the value when a new version will be released. We can confirm this in the `Dependency Dashboard`:

<p align="center">
  <img src="https://github.com/DucretJe/ducretje.github.io/assets/5384298/9652334c-d977-4118-b0d8-0e2c0ae836e7" />
</p>

Every time there is an update, the value changes and our script uses it to get the latest version of `starship`. This process must be repeated for each package of this kind.

As you can see, I have at least three others 🥷🏻

### Bump the packet manager handled packages

This is the trickiest part, and to be honest, I thought it was impossible for Renovate to bump some pinned versions of package manager-handled binaries. And well... I was wrong!

The trick is to use [Repology](https://repology.org/) as a data source.

```json
"regexManagers": [
...SNIPPED...
    {
      "fileMatch": [
        "^docker/.*Dockerfile$"
      ],
      "matchStrings": [
        "#*renovate: datasource=(?<datasource>.*?) depName=(?<depName>.*?) versioning=(?<versioning>.*?)\nENV .*_VER (?<currentValue>.*?)\n"
      ],
      "versioningTemplate": "{{#if versioning}}{{versioning}}{{else}}semver{{/if}}"
    }
 ],
```

We need to direct Renovate to our `Dockerfile` and then ask it to locate a comment that we added just before the variable definition. The variable definition should have a name that looks like `SOMETHING_VER`.

In the comment, we inform Renovate that we are using the `repology` data source, and specify the dependency name along with the repository to look into (for example, `debian_12/ca-certificates`).

We will use `loose` versioning everywhere, as advised in the [documentation](https://docs.renovatebot.com/modules/datasource/repology/).

This setup is a bit more involved, but it allows Renovate to monitor your packages and can be confirmed in the dashboard.

<p align="center">
  <img src="https://github.com/DucretJe/ducretje.github.io/assets/5384298/848abec0-4870-494f-b829-77acc916a3d6" />
</p>

It will stick to the version that we see [here](https://repology.org/project/ca-certificates/versions):

<p align="center">
  <img src="https://github.com/DucretJe/ducretje.github.io/assets/5384298/009fc43e-f9fe-4618-abfa-a71b6b50d5d6" />
</p>

This process must also be repeated for **each package** we install, it takes some time to do but it’s done once for all. We can see it as an investment.

### Setup renovate and the repository to allow auto-merge

To enable Renovate to function without waiting for our validation before merging a bump, we need to add some configuration.

```json
...SNIPPED...
	"automerge": true,
	"automergeType": "pr"
...SNIPPED...
```

To enable automerge in Renovate, we need to create pull requests (PRs) to trigger the mandatory tests, which only run on PRs. If the tests fail, the PRs cannot be auto-merged.

To make the tests mandatory, we will add branch protection to `main` in the following way:

<p align="center">
  <img src="https://github.com/DucretJe/ducretje.github.io/assets/5384298/74c27059-6b65-4be3-84d1-b9de0a8a05c8" />
</p>

I have two workflows with the name `test`:

- [KICS](https://kics.io/): I use this workflow to ensure that my code does not have any security issues. Here, I monitor the Dockerfile.
- Tests: I will explain more about this later.

The workflows themselves are not the subject of this post, so I won't describe them here.

Renovate will create a pull request and wait for the tests to pass. However, it may take several hours for the pull request to merge. In my case, it took approximately three hours.
{:.note title="💭 Note"}

## The Dockerfile

In Renovate's configuration, we mention this file several times. As you may have guessed, we will use variables to set the versions of our packages.

This post does not cover how a `Dockerfile` works; it will focus only on our two different types of packages:

- Manually installed packages
- Packages installed with a packet manager. Here, we are using a `debian` distribution, so we will use `apt-get`.

### Manually installed packages

Let's revisit the example I mentioned earlier, the `starship` package.

```docker
FROM debian:12.0

...SNIPPED...

# Github-releases
...SNIPPED...
ENV STARSHIP_VER 1.15.0
...SNIPPED...

# Install Starship
SHELL ["/bin/bash", "-c"]
RUN wget -q https://github.com/starship/starship/releases/download/v${STARSHIP_VER}/starship-x86_64-unknown-linux-gnu.tar.gz -O /tmp/starship.gz && \
    wget -q https://github.com/starship/starship/releases/download/v${STARSHIP_VER}/starship-x86_64-unknown-linux-gnu.tar.gz.sha256 -O /tmp/starship.sha256 && \
    SHA_ORIGIN_STARSHIP=$(cat /tmp/starship.sha256) && \
    SHA_DOWNLOADED_STARSHIP="$(sha256sum /tmp/starship.gz)" && \
    set -- $SHA_DOWNLOADED_STARSHIP && \
    SHA_DOWNLOADED_ARRAY=("$@") && \
	    if [ "$SHA_ORIGIN_STARSHIP" == "${SHA_DOWNLOADED_ARRAY[0]}" ]; then \
	    tar -xf "/tmp/starship.gz" -C /tmp/ && \
	    chmod +x "/tmp/starship" && \
	    mv "/tmp/starship" "/usr/bin/"; \
    else \
	    exit 1; \
    fi && \
    rm -f "/tmp/*.gz"
# False KICS positive in the following command triggering `Chown Flag Exists`, it's not an executable
# kics-scan ignore-line
COPY --chown=${USER}:${USER} starship.toml /home/${USER}/.config/starship.toml
COPY starship.toml /root/.config/starship.toml
SHELL ["/bin/sh", "-c"]
```

First, we declare the `STARSHIP_VER` variable in Renovate's configuration, currently set to `1.15.0`. However, this value will be updated by Renovate when a new version is released.

Next, we use this variable to download the binary and its checksum. We compute the checksum of the downloaded file and compare it with the value contained in the `starship.sha256` file to ensure that we do not have a corrupted package.

If the checksums match, we inflate the archive and move it to `/usr/bin`. Then, we perform some cleanup.

To ensure consistent behavior when using our normal user or `sudo su`, we add a configuration file in both the user's home directory and the root directory.

Please note that the shell is changed to bash for this part, as some commands may differ between shells and it is more convenient to use bash here.

### Packet manager

The example I used to demonstrate how to renovate with `ca-certificates` was quite straightforward. However, I encountered some difficulties with other packages. To illustrate this point, I will use two different packages:

- `ca-certificates`
- `ssh`

The difference between the two packages is simple: `ssh` uses the `epoch` and a `debian-revision`. The issue is that Renovate does not provide us with the `epoch` or the `debian-revision`.

Let's begin with the easier package:

```docker
FROM debian:12.0

...SNIPPED...

# Repology
...SNIPPED...
# renovate: datasource=repology depName=debian_12/ca-certificates versioning=loose
ENV CACERTIFICATES_VER 20230311
...SNIPPED...

# Install base packages
RUN apt-get update && apt-get install --no-install-recommends -y \
    ca-certificates=${CACERTIFICATES_VER} \
		...SNIPPED...
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
...SNIPPED...
```

The key point here is the comment just before the variable definition. As we saw in Renovate's regex, we point Renovate to the right package to monitor and where. Renovate will update the value automatically.

Note that several other packages are installed in the same block, which is why the `&&` is missing after `ca-certificates`.

This part is quite easy to understand: we use our variable to pin the version installed with `apt-get`.

Now let's move on to a more complicated package: `ssh`.

```docker
FROM debian:12.0

...SNIPPED...
# renovate: datasource=repology depName=debian_12/openssh versioning=loose
ENV SSH_VER 9.2p1
...SNIPPED...

# Install SSH
# False positive in HADOLINT, the version is pinned
# hadolint ignore=DL3008
RUN apt-get update && apt-get install --no-install-recommends -y \
    "?and(?exact-name(ssh),?version(:${SSH_VER}))" && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
...SNIPPED...
```

The beginning is the same, but notice that this version does not include either the `epoch` or the `debian-revision`. If you try to define your package using only the version, it will fail.

<script async id="asciicast-594096" src="https://asciinema.org/a/594096.js" data-autoplay="true" data-size="big" data-loop="1" data-rows="10"></script>

I managed to solve the issue with the `debian-revision` for other packages by adding a wildcard to it. However, we cannot use this approach for the `epoch` as well. For example, `apt install ssh=*:9.2p1*` won't work.

That's why I use a `search pattern` to find it.

## Tests

If we give Renovate too much freedom, it may merge an update that breaks the image. For example, the dependencies of package A may become incompatible with those of package B.

To address this issue, as previously mentioned, we will add tests to build the image and verify that the binaries can be used. We can use `make` to run these tests, but we won't cover how `make` works in this post.

```makefile
.PHONY: all build ssh starship

GREEN='\033[0;32m'
LGRAY='\033[0;37m'
NC='\033[0m'
ORANGE='\033[1;33m'
RED='\033[0;31m'

all: ssh starship

define test_command
	@echo ${ORANGE}Testing $(1) 🔬${LGRAY} && \
	docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v `pwd`:/rec --entrypoint $(1) test $(2); \
	if [ $$? = 0 ]; then \
		echo ${GREEN}$(1) OK ✅${NC}; \
	else \
		echo ${RED}$(1) test failed 🛑${NC}; \
		echo "1" > test-fail; \
	fi
endef

...SNIPPED...

build:
	@cd ../docker && \
	docker build -t test .

...SNIPPED...

ssh: build
	$(call test_command,ssh,-V)

starship: build
	$(call test_command,starship,--version)

...SNIPPED...

check:
	@if [ -f test-fail ]; then \
		rm test-fail; \
		echo ${RED} Some tests failed; \
		exit 1; \
	fi
```

Here, what we are doing is quite simple:

- We are trying to build our image.
- We test the binary by calling its version. If it succeeds, we can consider the binary as installed.

Of course, some binaries may need more tests to ensure they are installed and set up correctly, but I won't dig into this topic in this already-too-long post.

This test can be used locally and will also be used by GitHub Action to validate the PR.

## Wrap up

Through this exercise, I have learned several things:

- How to use Renovate with pinned versions of packages in a `Dockerfile`
- The Debian package version convention
- The use of `Make` functions

I hope my experience can be beneficial to you as well!
