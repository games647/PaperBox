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

[Reference](https://github.com/GoogleContainerTools/jib/blob/master/docs/faq.md#i-want-to-containerize-a-jar)

**Note**: The last point is currently not implemented, because this project currently uses Paperclip instead of 
something custom. Ideal would be the usage of a JIB Gradle task to automatically extract all necessary dependencies
and include a modified version of `paperweight` to create a diff, but skipping the jar and shadowing step. It seems
there is no such solution as of now.

## Features

* Offline
  * All necessary files to run the server are already included in the image
  * Including libraries, vanilla server (patching soure) and server implementation
  * No more downloads are necessary
* Binary patching of the server implementation on startup (`paperclip`)
  * Faster than running through BuildTools (no decompile, patch, recompile)
  * Distributing the server implementation could be against the GPL
* Minimal: only 300 MiB vs 450 MiB (`felixklauke/paperspigot`) or 800 MiB (`itzg/minecraft-server`)
  * No unnecessary packages like `dos2unix` or `nano` ([ref](https://github.com/itzg/docker-minecraft-server/blob/8f8acc40f5a779c8cc3b0de4909a0e41894d7218/Dockerfile#L21))
  * Based on the distroless base image
    * Reduced attack surface and image size
    * No shell or package manager
* Rootless user (is that really a feature? it should be standard)
* Cache-optimized layer layout
* Tons of metadata labels
  * Including base image digest label
* Required EULA acceptance - make this decision transparent to the user

## Concept

The idea of this project relies on the fact that layers in images are cached and can be re-used. If a layer changes,
itself and all following layers needs to be rebuilt. Therefore, it's recommended to place content that frequently
changes at the end. See the layout below.

Additionally, to the improved layer layout the GPL compliance needs to addressed. The server implementation violates
this license. Distributing it, also seems not be allowed. However, binary patching at the users seems to be allowed
like in similar projects [BuildTools](https://www.spigotmc.org/wiki/buildtools/) or
[paperclip](https://github.com/PaperMC/Paperclip).

### Implementation

The implementation can be found in the GitHub actions file [here](.github/workflows/build.yml). It can be summarized in
the following steps:

1. Download Paper
2. Create a Paperclip version
3. Extract the jar version
4. Separate the folders into different layers
   1. Earlier version used `layers.idx` from 
   [Spring](https://spring.io/blog/2020/08/14/creating-efficient-docker-images-with-spring-boot-2-3)
   2. However, this doesn't support including the vanilla jar in an arbitrary location
6. Invoke the JIB-CLI tool to build the container according to this [build file](abstract-jib.yaml)

Creating the following image layer layout:

| Layer | Implementation                     | Ideal?                                      |
|-------|------------------------------------|---------------------------------------------|
| -     | Base Image                         | Base image                                  |
| 1     | Paperclip libraries                | Paperclip libraries                         |
| 2     | Paperclip classes                  | Paperclip classes                           |
| 3     | Vanilla server                     | Vanilla server                              |
| 4     | Server libraries (incl. Paper-API) | Libraries                                   |
| 5     | Server patch                       | Snapshot libraries                          |
| 6     | -                                  | Module dependencies (Paper-API, Mojang-API) | 
| 7     | -                                  | Static files                                |
| 8     | -                                  | Sever patch                                 |

Meanwhile, the [recommended](https://github.com/GoogleContainerTools/jib/blob/master/docs/faq.md#how-are-jib-applications-layered) 
setup by JIB includes the right side, because snapshot libraries are updated more often.

## Limitations

The chosen approach has a few limitations and Anti-Patterns, namely:

* Patching the server is stateful

### Distributing Mojang vanilla server

Distributing the Mojang vanilla server in an extra layer, means we are always including the full, original server. This
is inefficient, because changes in Paper's server implementation remove, modify parts of the vanilla server making
these unreachable.

### Patching overhead on server startup

Of course the patching process, necessary for GPL compliance, also adds a delay to the server startup. However, this
overhead should be drastically reduced, because all necessary files are locally available and only the server
implementation needs to be patched.

However, Paperclip supports patch-only start, which you could use to warm up the container. You can specify the
parameter using `-Dpaperclip.patchonly=true`.
[ref](https://github.com/PaperMC/Paperclip/blob/dcd86a9faf4ee82b434bcf26a8e4b2dd4eb39c87/java17/src/main/java/io/papermc/paperclip/Paperclip.java)

## Alternative Tools

* [Dockerfile Maven](https://github.com/spotify/dockerfile-maven)
* [Buildpacks](https://github.com/paketo-buildpacks/executable-jar)
    * Includes advanced features like:
      * rebasing of base images
      * SBOM
      * Memory Calculator on startup
      * `MALLOC_ARENA_MAX` according to active processor count to help memory consumption

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

## Running

Running it, requires accepting the EULA. This can be specified using environment variables:
```shell
docker run --env JDK_JAVA_OPTIONS="-Dcom.mojang.eula.agree=true" ghcr.io/games647/paperclip:1.18.1 ...
podman run --env JDK_JAVA_OPTIONS="-Dcom.mojang.eula.agree=true" ghcr.io/games647/paperclip:1.18.1 ...
```

### Configuring

* Memory settings: Since Java 10, the JVM is aware of cgroup limits. 
  See `podman run --memory` or `docker run --memory`.
  [ref](https://www.atamanroman.dev/articles/usecontainersupport-to-the-rescue/)
* For specific settings [Java Memory Calculator](https://github.com/cloudfoundry/java-buildpack-memory-calculator) is
  recommended, but specific to your server setup like plugins, etc.
  * `$ java-buildpack-memory-calculator --total-memory 2G --loaded-class-count 500 --thread-count 20`
  * Result: `-XX:MaxDirectMemorySize=10M -XX:MaxMetaspaceSize=16503K -XX:ReservedCodeCacheSize=240M -Xss1M -Xmx1804168K`
* JVM Flags. The project includes the [aikar JVM](ttps://paper.readthedocs.io/en/latest/server/aikar-flags.html) flags
  by default using the `JDK_JAVA_OPTIONS` environment variable. If you want to modify or remove flags, use
  `--env JDK_JAVA_OPTIONS=...` to override it. However, if you want to add additional flags use the environment variable
  from above like `--env JDK_JAVA_OPTIONS="-Dcom.mojang.eula.agree=true -Xmx1G"`.
* Similar remote debugging can be enabled using the following parameters:
  `--env JDK_JAVA_OPTIONS="-Dcom.mojang.eula.agree=true -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=localhost:5005"`

### Useful projects

* [Auto pause](https://github.com/timvisee/lazymc): Automatically stops or pauses the server process if the server is
  idle. For container projects, this could be realized using a sidecar pattern. However, this would require container
  engine access, which is not recommended for security, or using an additional service inside the container.
  **Open for any suggestions**

## Tools used

Besides, the obvious JIB-Tool:

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
