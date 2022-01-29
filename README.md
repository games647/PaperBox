# PaperBox

## Description

Build the Minecraft Paper server software using an optimized container layout format. The
[JIB-Tool](https://github.com/GoogleContainerTools/jib) builds OCI or Docker images by separating library dependencies 
into a different layer than the actual source code (here: Spigot/Paper changes). This improves layer cache utilization,
because libraries are less likely to be updated than the source code. In the end, this provides the following benefits:

* Reduced network traffic: OCI images are downloaded by layer. If the library layer is already present and there are no
  changes, then it doesn't need to be downloaded.
* Reduced disk space: Same as above. Layers are re-used if they are the same and do not use additional space.
* Improved build time: 
  * No fat jar/shadowing/shading of libraries: Including the libraries is now 
  * No packaging into jar: We can run the class files directly from the file system with an adjusted classpath. 
    This removes the packaging into jar step.

**Note**: The last point is currently not implemented, because this project currently uses Paperclip instead of 
something custom. 

## Features

* Offline
  * All necessary files to run the server are already included in the image
  * Specifically it contains the all libraries, the vanilla server and the diff files to get the patched server 
  implementation.
* Minimal container: only 300 MiB vs 450 MiB (`felixklauke/paperspigot`) or 800 MiB (`itzg/minecraft-server`)
  * No unnecessary packages like `dos2unix` or ``nano` ([ref](https://github.com/itzg/docker-minecraft-server/blob/8f8acc40f5a779c8cc3b0de4909a0e41894d7218/Dockerfile#L21))
  * Based on the distroless base image
* Rootless user (is that really a feature? it should be standard)
* Cache-optimized layer layout
* Required EULA acceptance - make this decision transparent to the user
* Binary patching of the server implementation on startup
  * Distributing the server implementation could be against the GPL

## Running

Running it, requires accepting the EULA. This can be specified using environment variables:
```shell
docker run -e JDK_JAVA_OPTIONS="-Dcom.mojang.eula.agree=true" ...
podman run -e JDK_JAVA_OPTIONS="-Dcom.mojang.eula.agree=true" ...
```

### Configuring

* Memory settings: Since Java 10, the JVM is aware of cgroup limits. So it should be preferred to configure
  the container memory limits. See `podman run --memory` or `docker run --memory`.
  [ref](https://www.atamanroman.dev/articles/usecontainersupport-to-the-rescue/)
* JVM Flags. The project includes the [aikar JVM](ttps://paper.readthedocs.io/en/latest/server/aikar-flags.html) flags
  by default using the `JDK_JAVA_OPTIONS` environment variable. If you want to modify or remove flags, use
  `-e JDK_JAVA_OPTIONS=...` to override it. However, if you want to add additional flags use the environment variable
  from above like `-e JDK_JAVA_OPTIONS="-Dcom.mojang.eula.agree=true -Xmx1G"`.
* Similar remote debugging can be enabled using the following parameters:
  `-e JDK_JAVA_OPTIONS="-Dcom.mojang.eula.agree=true -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=localhost:5005`

### Useful projects

* [Auto pause](https://github.com/timvisee/lazymc): Automatically stops or pauses the server process if the server is
  idle. For container projects, this could be realized using a sidecar pattern. However, this would require container
  engine access, which is not recommended for security, or using an additional service inside the container.
  **Open for any suggestions**

## Concept

The idea of this project relies on the fact that layers in images are cached and can be re-used. If a layer changes,
itself and all following layers needs to be rebuilt. Therefore, it's recommended to place content that frequently
changes at the end. See the layout below.

Additionally, to the improved layer layout the GPL compliance needs to addressed. The server implementation violates
this license. Distributing it, also seems not be allowed. However, binary patching at the users seems to be allowed
like in similar projects [BuildTools](https://www.spigotmc.org/wiki/buildtools/) or
[paperclip](https://github.com/PaperMC/Paperclip).

## Image layer layout

Concept currently only:
1. Base image (current: `gcr.io/distroless/java17-debian11:nonroot` for security and minimalism)
2. Libraries
3. Snapshot libraries (are more likely to change)
4. Mojang vanilla server - updated between Minecraft versions
5. Paper API and MojangAPI - available without restrictions
6. Binary patch vanilla server with server implementation changes

## Limitations

The chosen approach has a few limitations as well:

### Distributing Mojang vanilla server

Distributing the Mojang vanilla server in an extra layer, means we are always including the full, original server. This
is inefficient, because changes in Paper's server implementation remove, modify parts of the vanilla server making
these unreachable.

### Patching overhead on server startup

Of course the patching process, necessary for GPL compliance, also adds a delay to the server startup. However, this
overhead should be drastically reduced, because all necessary files are locally available and only the server
implementation needs to be patched.

## Differences to other OCI/Container approaches

Besides, utilizing a more optimized layer layout:

### itzg/minecraft-server

The project at [itzg/minecraft-server](https://hub.docker.com/r/itzg/minecraft-server) downloads the server software
at container startup. This increases the container startup time. Containers should be ready to start directly.
(Note: The patching process implemented in this project also has a delay) Furthermore, it downloads a file from a 
foreign source (by default: https://getbukkit.org/). In a very strict setup, this affects the security policies. First, 
it requires allowing an otherwise unnecessary outgoing connection (e.g. allow list policies) and trusting a 
foreign source. There no verification process involved, that would detect changed server jar on the upstream server 
like checksum. 

### FelixKlauke/paperspigot-docker

[FelixKlauke/paperspigot-docker](https://github.com/FelixKlauke/paperspigot-docker) builds the Paper server software
and provides the server jar in the image. This means that there is no download or patching process involved when the
container is created and started. However, this means that we are distributing a modified server software, which is
against the GPL. Latter required us to create solutions like [BuildTools](https://www.spigotmc.org/wiki/buildtools/) or
[paperclip](https://github.com/PaperMC/Paperclip) to build server on our own or binary patch it.

Furthermore, the EULA is automatically accepted, which hides it from the actual users. The process should be opt-in
explicitly to make it transparent to users.

## Tools used

### nektos/act

[nektos/act](https://github.com/nektos/act) allows running GitHub actions locally. Cloning the Paper
repository requires configuring a GitHub access token. Generate one [here](https://github.com/settings/tokens/new) and
set it using `export GITHUB_TOKEN=ghp_[...]`. Note tha

Docker:
```shell
act --reuse --bind --secret GITHUB_TOKEN="$GITHUB_TOKEN"
```

Podman:

Podman requires additional setup. Assuming podman is configured it rootless mode and systemd is present:
```shell
# Enable the podman socket using systemd
systemctl enable --now --user podman.socket
# Emulate a Docker socket
export DOCKER_HOST=unix://"$XDG_RUNTIME_DIR"/podman/podman.sock
act --reuse --bind --container-daemon-socket "$XDG_RUNTIME_DIR"/podman/podman.sock --secret GITHUB_TOKEN="$GITHUB_TOKEN"
```

#### Arguments

* `--reuse`: Keep container state. Prevents additional rebuilds
* `--secret`: Required to download the Paper repository in the checkout step
* `--bind`: Bind the container to the working directory

#### wagoodman/dive

[wagoodman/dive](https://github.com/wagoodman/dive) inspect the contents of the image for each layer. Inspect the
separation process. Example:

```shell
dive podman://localhost/IMAGE
```
