# PaperBox

## Description

Build the Minecraft Paper server software using an optimized container layout format. The
[JIB-Tool](https://github.com/GoogleContainerTools/jib) builds OCI images by separating library dependencies into
a different layer than the actual source code (here: Spigot/Paper changes). This improves layer cache utilization,
because libraries are less likely to be updated than the source code. In the end, this provides the following benefits:

* Reduced network traffic: OCI images are downloaded by layer. If the library layer is already present and there are no
  changes, then it doesn't need to be downloaded.
* Reduced disk space: Same as above
* Improved image build time: First the existing library layer can be used and second there is no need to package the
  project into a jar file. As long the project is compiled into class files, it's ready to startup.
* No fat jar: With Jib it's unnecessary to shade/shadow libraries into a big jar file. The classpath inside the image
  will find all necessary libraries.

## Differences to other OCI/Container approaches

Besides, utilizing a more optimized layer layout:

### itzg/minecraft-server

The project at [itzg/minecraft-server](https://hub.docker.com/r/itzg/minecraft-server) downloads the server software
at container startup. This increases the container startup time. Containers should be ready to start directly.
Furthermore, it downloads a file from a foreign source (by default: https://getbukkit.org/). In a very strict setup,
this affects the security policies. First, it requires allowing an otherwise unnecessary outgoing connection
(e.g. allow list network policy) and trusting a foreign source. There no verification process involved, that would
detect changed server jar on the upstream server (like checksum to expect a certain build). This would make it also
harder to debug it, because they are not reproducible.

### FelixKlauke/paperspigot-docker

[FelixKlauke/paperspigot-docker](https://github.com/FelixKlauke/paperspigot-docker) builds the Paper server software
and provides the server jar in the image. This means that there no download or patching process involved when the
container is created and started. However, this means that we are distributing a modified server software, which is
against the GPL. Latter required us to create solutions like [BuildTools](https://www.spigotmc.org/wiki/buildtools/) or
[paperclip](https://github.com/PaperMC/Paperclip) to build server on our own or binary patch it.

## Tools used

### nektos/act

[nektos/act](https://github.com/nektos/act) allows running GitHub actions locally. Cloning the Paper
repository requires configuring a GitHub access token. Generate one [here](https://github.com/settings/tokens/new) and
set it using `export GITHUB_TOKEN=ghp_[...]`. Note tha

Docker:
```shell
act --reuse --bind --secret GITHUB_TOKEN=$GITHUB_TOKEN
```

Podman:

Podman requires additional setup. Assuming podman is configured it rootless mode and systemd is present:
```shell
# Enable the podman socket using systemd
systemctl enable --now --user podman.socket
# Emulate a Docker socket
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/podman/podman.sock
act --reuse --bind --container-daemon-socket $XDG_RUNTIME_DIR/podman/podman.sock --secret GITHUB_TOKEN=$GITHUB_TOKEN
```

#### Arguments

* `--reuse`: Keep container state. Prevents additional rebuilds
* `--secret`: Required to download the Paper repository in the checkout step
* `--bind`: Bind the container to the working directory

#### wagoodman/dive

[wagoodman/dive](https://github.com/wagoodman/dive) inspect the contents of the image for each layer. Inspect the
separation process.
