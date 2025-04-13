# Guide to Build Homebrew from Dockerfile

This guide provides detailed steps to build and use Homebrew from a Dockerfile. Follow these instructions to create a Docker image that includes Homebrew and to run Homebrew commands inside a Docker container.

## Prerequisites

Before you begin, ensure you have the following installed on your machine:
- Docker: [Download and install Docker](https://www.docker.com/products/docker-desktop)

## Creating the Dockerfile

Create a file named `Dockerfile` in your project directory and add the following content to it:

```dockerfile
ARG version=24.10
# version is passed through by Docker.
# shellcheck disable=SC2154
FROM ubuntu:"${version}"
ARG DEBIAN_FRONTEND=noninteractive

# Deterministic UID (first user). Helps with docker build cache
ENV USER_ID=1000
# Delete the default ubuntu user & group UID=1000 GID=1000 in Ubuntu 23.04+
# that conflicts with the linuxbrew user
RUN touch /var/mail/ubuntu && chown ubuntu /var/mail/ubuntu && userdel -r ubuntu; true

# We don't want to manually pin versions, happy to use whatever
# Ubuntu thinks is best.
# hadolint ignore=DL3008

# `gh` installation taken from https://github.com/cli/cli/blob/trunk/docs/install_linux.md#debian-ubuntu-linux-raspberry-pi-os-apt
# /etc/lsb-release is checked inside the container and sets DISTRIB_RELEASE.
# We need `[` instead of `[[` because the shell is `/bin/sh`.
# shellcheck disable=SC1091,SC2154,SC2292
RUN apt-get update \
  && apt-get install -y --no-install-recommends software-properties-common gnupg-agent \
  && if [ "$(uname -m)" != aarch64 ]; then add-apt-repository -y ppa:git-core/ppa; fi \
  && apt-get update \
  && apt-get install -y --no-install-recommends \
  acl \
  bzip2 \
  ca-certificates \
  curl \
  file \
  fonts-dejavu-core \
  g++ \
  gawk \
  git \
  gpg \
  less \
  locales \
  make \
  netbase \
  openssh-client \
  patch \
  sudo \
  unzip \
  uuid-runtime \
  tzdata \
  jq \
  && if [ "$(. /etc/lsb-release; echo "${DISTRIB_RELEASE}" | cut -d. -f1)" -ge 22 ]; then apt-get install -y --no-install-recommends skopeo; fi \
  && mkdir -p /etc/apt/keyrings \
  && chmod 0755 /etc /etc/apt /etc/apt/keyrings \
  && curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | tee /etc/apt/keyrings/githubcli-archive-keyring.gpg >/dev/null \
  && chmod go+r /etc/apt/keyrings/githubcli-archive-keyring.gpg \
  && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | tee /etc/apt/sources.list.d/github-cli.list >/dev/null \
  && apt-get update \
  && apt-get install -y --no-install-recommends gh \
  && apt-get remove --purge -y software-properties-common \
  && apt-get autoremove --purge -y \
  && rm -rf /var/lib/apt/lists/* \
  && sed -i -E 's/^(USERGROUPS_ENAB\s+)yes$/\1no/' /etc/login.defs \
  && localedef -i en_US -f UTF-8 en_US.UTF-8 \
  && useradd -u "${USER_ID}" --create-home --shell /bin/bash --user-group linuxbrew \
  && echo 'linuxbrew ALL=(ALL) NOPASSWD:ALL' >>/etc/sudoers \
  && su - linuxbrew -c 'mkdir ~/.linuxbrew'

USER linuxbrew
COPY --chown=linuxbrew:linuxbrew . /home/linuxbrew/.linuxbrew/Homebrew
ENV PATH="/home/linuxbrew/.linuxbrew/bin:/home/linuxbrew/.linuxbrew/sbin:${PATH}" \
  XDG_CACHE_HOME=/home/linuxbrew/.cache
WORKDIR /home/linuxbrew


RUN --mount=type=cache,target=/tmp/homebrew-core,uid="${USER_ID}",sharing=locked \
  # Clone the homebrew-core repo into /tmp/homebrew-core or pull latest changes if it exists
  git clone https://github.com/homebrew/homebrew-core /tmp/homebrew-core || { cd /tmp/homebrew-core && git pull; } \
  && mkdir -p /home/linuxbrew/.linuxbrew/Homebrew/Library/Taps/homebrew/homebrew-core \
  && cp -r /tmp/homebrew-core /home/linuxbrew/.linuxbrew/Homebrew/Library/Taps/homebrew/


RUN --mount=type=cache,target=/home/linuxbrew/.cache,uid="${USER_ID}" \
  --mount=type=cache,target=/home/linuxbrew/.bundle,uid="${USER_ID}" \
  mkdir -p \
  .linuxbrew/bin \
  .linuxbrew/etc \
  .linuxbrew/include \
  .linuxbrew/lib \
  .linuxbrew/opt \
  .linuxbrew/sbin \
  .linuxbrew/share \
  .linuxbrew/var/homebrew/linked \
  .linuxbrew/Cellar \
  && ln -s ../Homebrew/bin/brew .linuxbrew/bin/brew \
  && git -C .linuxbrew/Homebrew remote set-url origin https://github.com/Homebrew/brew \
  && git -C .linuxbrew/Homebrew fetch origin \
  && HOMEBREW_NO_ANALYTICS=1 HOMEBREW_NO_AUTO_UPDATE=1 brew tap --force homebrew/core \
  && brew install-bundler-gems --groups=all \
  && brew cleanup \
  && { git -C .linuxbrew/Homebrew config --unset gc.auto; true; } \
  && { git -C .linuxbrew/Homebrew config --unset homebrew.devcmdrun; true; } \
  && touch .linuxbrew/.homebrewdocker
```

## Building the Docker Image

Open a terminal and navigate to the directory containing the `Dockerfile`. Run the following command to build the Docker image:

```sh
docker build -t homebrew:latest .
```

This command will create a Docker image named `homebrew` with the tag `latest`.

## Running a Container from the Image

Once the image is built, you can run a container from it using the following command:

```sh
docker run -it homebrew:latest
```

This command starts a new container in interactive mode and opens a bash shell.

## Using Homebrew in the Container

Inside the running container, you can use Homebrew commands as usual. Here are some examples:

- **Check Homebrew version:**
  ```sh
  brew --version
  ```

- **Update Homebrew and installed formulae:**
  ```sh
  brew update
  brew upgrade
  ```

- **Install a package with Homebrew:**
  ```sh
  brew install wget
  ```

## Detaching and Reattaching to the Container

To detach from the running container without stopping it, use the following key combination:
- **CTRL + P, then CTRL + Q** or ```exit```

To list running containers and find the container ID or name:
```sh
docker ps
```

To reattach to the running container:
```sh
docker attach <container_id_or_name>
```

## Stopping and Restarting the Container

To stop the running container:
```sh
docker ps -as | grep <search_term> | awk '{print $1}' | xargs docker stop
```
Replace `<search_term>` with a relevant term to filter the container you wish to stop.

To restart the stopped container:
```sh
docker start <container_id_or_name>
```

To reattach to the restarted container:
```sh
docker attach <container_id_or_name>
```

## Using `docker exec` to Access a Running Container

If you need to execute commands inside a running container without attaching to it, you can use the `docker exec` command. First, list the running containers:

```sh
docker ps -as | grep <search_term> | awk '{print $1}'
```
Replace `<search_term>` with a relevant term to filter the container you wish to access.

Then, use the container ID to run a command inside the container:

```sh
docker exec -it <container_id> /bin/bash
```

## Additional Resources

For more information on Docker and Homebrew, refer to the following resources:
- [Docker Documentation](https://docs.docker.com/)
- [Homebrew Documentation](https://docs.brew.sh/)

By following this guide, you should be able to build and use Homebrew from a Dockerfile effectively. If you encounter any issues or have questions, feel free to refer to the official documentation or seek assistance from the community.
