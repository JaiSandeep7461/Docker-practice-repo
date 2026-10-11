# Dockerfile Instructions: Demonstration and Testing Guide

This README demonstrates the 10 Dockerfile instructions covered in the learning session. Each section includes a Dockerfile example, build command, run/test command, and expected result.

## Prerequisites

- Docker Desktop or Docker Engine is installed and running.
- Run commands from the directory containing the relevant `Dockerfile`.
- Each example uses a separate image tag to avoid confusion between demonstrations.

Check Docker is running:

```bash
docker --version
docker info
```

> **Windows Git Bash note:** Git Bash can automatically convert Linux-style paths in some commands. If a command that includes a Linux path behaves unexpectedly, try prefixing it with `MSYS_NO_PATHCONV=1`, or run the command from an interactive container shell.

---

## 1. `FROM` — Choose a Base Image

`FROM` specifies the base image on which the new image is built.

### Dockerfile

```dockerfile
FROM ubuntu:24.04
CMD ["cat", "/etc/os-release"]
```

### Build

```bash
docker build -t from-demo:1.0 .
```

### Test

```bash
docker run --rm from-demo:1.0
```

### Expected result

The output shows Ubuntu operating system details, including its release information.

**Key point:** `FROM` selects a starting image. It does not mean the image contains every package that would be installed on a full Ubuntu virtual machine.

---

## 2. `WORKDIR` — Set the Working Directory

`WORKDIR` sets the working directory for subsequent Dockerfile instructions such as `RUN`, `COPY`, `CMD`, and `ENTRYPOINT`. Docker creates the directory if it does not already exist.

### Dockerfile

```dockerfile
FROM ubuntu:24.04
WORKDIR /app
CMD ["pwd"]
```

### Build

```bash
docker build -t workdir-demo:1.0 .
```

### Test

```bash
docker run --rm workdir-demo:1.0
```

### Expected result

```text
/app
```

**Key point:** Without a `WORKDIR` instruction, the default working directory for this Ubuntu image is `/`.

---

## 3. `COPY` — Copy Files into the Image

`COPY` copies files from the build context (usually the directory supplied as `.` to `docker build`) into the image.

### Create a test file

Create a file named `app.txt` beside the Dockerfile with this content:

```text
Hello from the copied file!
```

### Dockerfile

```dockerfile
FROM ubuntu:24.04
WORKDIR /app
COPY app.txt .
CMD ["cat", "app.txt"]
```

### Build

Run this command from the directory containing both `Dockerfile` and `app.txt`:

```bash
docker build -t copy-demo:1.0 .
```

### Test

```bash
docker run --rm copy-demo:1.0
```

### Expected result

```text
Hello from the copied file!
```

**Key points:**
- The final `.` in `COPY app.txt .` means the current working directory inside the image, `/app`.
- The source file must be inside the build context and not excluded by `.dockerignore`.

---

## 4. `RUN` — Execute Commands While Building the Image

`RUN` executes a command during the image build. The result of filesystem changes is saved in an image layer.

### Dockerfile

```dockerfile
FROM ubuntu:24.04
WORKDIR /app
RUN touch build-created.txt
CMD ["ls", "-l", "/app"]
```

### Build

```bash
docker build -t run-demo:1.0 .
```

### Test

```bash
docker run --rm run-demo:1.0
```

### Expected result

The listing includes `build-created.txt`.

**Key point:** The file is created during the build, so it is part of the image. This differs from a file created only in a running container's writable layer, which is lost when that container is removed.

---

## 5. `ARG` — Define a Build-Time Variable

`ARG` defines a variable that can be used during image building. It is not automatically available as an environment variable when the container runs.

### Dockerfile

```dockerfile
FROM ubuntu:24.04
ARG APP_VERSION=1.0
CMD ["sh", "-c", "echo APP_VERSION=${APP_VERSION:-not-set}"]
```

### Build

```bash
docker build -t arg-demo:1.0 .
```

To override the build-time value:

```bash
docker build --build-arg APP_VERSION=2.0 -t arg-demo:2.0 .
```

### Test

```bash
docker run --rm arg-demo:1.0
```

### Expected result

```text
APP_VERSION=not-set
```

The `ARG` value was available during the build, but it was not automatically passed into the running container's environment.

**Key point:** If an argument is needed as a runtime environment variable, explicitly transfer it with `ENV`, for example `ARG APP_VERSION=1.0` followed by `ENV APP_VERSION=${APP_VERSION}`. Do not use `ARG` or `ENV` to store secrets.

---

## 6. `ENV` — Set an Environment Variable

`ENV` sets an environment variable in the image. The variable is available to processes started in containers created from that image.

### Dockerfile

```dockerfile
FROM ubuntu:24.04
ENV APP_ENV=development
CMD ["sh", "-c", "echo APP_ENV=$APP_ENV"]
```

### Build

```bash
docker build -t env-demo:1.0 .
```

### Test the default value

```bash
docker run --rm env-demo:1.0
```

Expected output:

```text
APP_ENV=development
```

### Override the value at runtime

```bash
docker run --rm -e APP_ENV=production env-demo:1.0
```

Expected output:

```text
APP_ENV=production
```

**Key point:** The runtime override applies to that container. It does not modify the original image's default.

---

## 7. `EXPOSE` — Document the Container Port

`EXPOSE` documents the port on which an application is expected to listen inside the container. It does **not** publish the port to the host by itself.

### Dockerfile

```dockerfile
FROM ubuntu:24.04
EXPOSE 8080
CMD ["sh", "-c", "echo 'This image documents port 8080'; sleep 3600"]
```

### Build

```bash
docker build -t expose-demo:1.0 .
```

### Test the exposed-port metadata

```bash
docker image inspect expose-demo:1.0 --format '{{json .Config.ExposedPorts}}'
```

Expected output includes:

```text
{"8080/tcp":{}}
```

### Run the container

```bash
docker run -d --name expose-demo-container expose-demo:1.0
```

Inspect the running container:

```bash
docker ps --filter name=expose-demo-container
docker port expose-demo-container
```

`docker port` will not show a host mapping because no port was published.

### Publish a port to the host

The example above does not run a web server; it only demonstrates port metadata. To publish a port, an application must actually listen on that container port. The mapping syntax is:

```bash
docker run --rm -p 8080:8080 IMAGE_NAME
```

This maps host port `8080` to container port `8080`; it does not make a non-listening application respond.

### Clean up

```bash
docker rm -f expose-demo-container
```

---

## 8. `USER` — Choose the User for Subsequent Instructions

`USER` sets the user used by subsequent `RUN` instructions and the default user for the container's runtime process. The account must exist before it is selected.

### Dockerfile

```dockerfile
FROM ubuntu:24.04
RUN useradd --create-home appuser
USER appuser
CMD ["whoami"]
```

### Build

```bash
docker build -t user-demo:1.0 .
```

### Test

```bash
docker run --rm user-demo:1.0
```

### Expected result

```text
appuser
```

**Key points:**
- `RUN useradd --create-home appuser` creates the account.
- `USER appuser` switches to that account for subsequent instructions and the default runtime process.
- Running applications as a non-root user follows the principle of least privilege and can reduce risk.

---

## 9. `ENTRYPOINT` — Define the Main Executable

`ENTRYPOINT` specifies the main executable that runs when the container starts. With the exec form, additional arguments supplied to `docker run` are appended to the executable and its configured arguments.

### Dockerfile

```dockerfile
FROM ubuntu:24.04
ENTRYPOINT ["echo", "Docker"]
```

### Build

```bash
docker build -t entrypoint-demo:1.0 .
```

### Test the default behavior

```bash
docker run --rm entrypoint-demo:1.0
```

Expected output:

```text
Docker
```

### Pass additional runtime arguments

```bash
docker run --rm entrypoint-demo:1.0 is awesome
```

Expected output:

```text
Docker is awesome
```

**Key point:** With this exec-form `ENTRYPOINT`, runtime arguments are appended. The configured executable remains `echo`.

---

## 10. `CMD` — Provide the Default Command or Arguments

`CMD` provides a default command and arguments, or default arguments for an `ENTRYPOINT`. Runtime arguments can replace the `CMD` defaults.

### Example A: `CMD` on its own

Dockerfile:

```dockerfile
FROM ubuntu:24.04
CMD ["echo", "Hello from Docker"]
```

Build:

```bash
docker build -t cmd-demo:1.0 .
```

Test:

```bash
docker run --rm cmd-demo:1.0
```

Expected output:

```text
Hello from Docker
```

**Important:** With this `CMD` alone, a runtime argument replaces the whole default command and argument list. For example, `docker run --rm cmd-demo:1.0 echo` runs `echo` with no configured message.

### Example B: Combine `ENTRYPOINT` and `CMD`

Dockerfile:

```dockerfile
FROM ubuntu:24.04
ENTRYPOINT ["echo"]
CMD ["Hello"]
```

Build:

```bash
docker build -t cmd-entrypoint-demo:1.0 .
```

Test the default argument:

```bash
docker run --rm cmd-entrypoint-demo:1.0
```

Expected output:

```text
Hello
```

Override the default argument:

```bash
docker run --rm cmd-entrypoint-demo:1.0 Docker
```

Expected output:

```text
Docker
```

**Key point:** `ENTRYPOINT` stays as `echo`, while the runtime argument `Docker` replaces the default `CMD` argument `Hello`.

---

## Quick Reference

| Instruction | Main purpose | When it takes effect |
|---|---|---|
| `FROM` | Select the base image | Build |
| `WORKDIR` | Set the working directory | Build and runtime defaults |
| `COPY` | Copy files into the image | Build |
| `RUN` | Execute build commands | Build |
| `ARG` | Define a build-time variable | Build |
| `ENV` | Set environment variables | Image build and runtime |
| `EXPOSE` | Document an expected container port | Image metadata |
| `USER` | Select the execution user | Subsequent build steps and runtime |
| `ENTRYPOINT` | Define the main executable | Runtime |
| `CMD` | Set default command/arguments | Runtime |

## Useful Docker Commands

```bash
# List images
docker images

# List running containers
docker ps

# List all containers, including stopped ones
docker ps -a

# View logs from a container
docker logs CONTAINER_NAME_OR_ID

# Inspect image configuration
docker image inspect IMAGE_NAME

# Remove an image (only after containers no longer depend on it)
docker image rm IMAGE_NAME
```

## Demonstration Tips

1. Keep each demonstration in its own folder, or replace the Dockerfile and rebuild with the image tag specified in that section.
2. For the `COPY` example, create `app.txt` in the same directory as the Dockerfile before building.
3. Use the same image tag in the build and run commands.
4. Use `docker run --rm` for short-lived examples; it removes the container after it exits, but does not remove the image.
5. `EXPOSE` alone does not publish a port. Use `-p HOST_PORT:CONTAINER_PORT` and run an application that listens on the container port to test connectivity.
6. Prefer exec-form JSON arrays such as `CMD ["echo", "Hello"]` and `ENTRYPOINT ["echo"]` for predictable argument handling.
