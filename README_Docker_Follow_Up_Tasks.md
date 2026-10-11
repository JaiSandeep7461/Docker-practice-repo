# Docker Follow-Up Practical Tasks

A hands-on roadmap to practise Dockerfile instructions after completing the introductory Dockerfile README.

These tasks deliberately focus on Docker images, Dockerfiles, containers, runtime configuration, troubleshooting, and port publishing. **Docker networking concepts and Docker volumes are excluded.**

## Prerequisites

- Docker Desktop or Docker Engine installed and running.
- A terminal such as PowerShell, Command Prompt, or Git Bash.
- Basic familiarity with `docker build` and `docker run`.
- Create a separate folder for each task, or replace the Dockerfile and rebuild using the image tag specified in that task.
- Unless a task says otherwise, run commands from the directory containing the `Dockerfile`.

Check Docker:

```bash
docker --version
docker info
```

### Windows Git Bash note

Git Bash may rewrite Linux-style paths in some commands. If a command containing a Linux path behaves unexpectedly, try prefixing it with `MSYS_NO_PATHCONV=1`, or run the command from an interactive container shell.

---

# Task 1 — Compare Base Images

**Goal:** Understand that different base images provide different operating-system userlands, utilities, and image sizes.

## 1.1 Ubuntu image

Create a `Dockerfile`:

```dockerfile
FROM ubuntu:24.04
CMD ["sh", "-c", "cat /etc/os-release; echo 'Available shell:'; command -v bash; echo 'Package manager:'; command -v apt"]
```

Build and run:

```bash
docker build -t task1-ubuntu:1.0 .
docker run --rm task1-ubuntu:1.0
```

## 1.2 Alpine image

Replace the Dockerfile with:

```dockerfile
FROM alpine:3.22
CMD ["sh", "-c", "cat /etc/os-release; echo 'Available shell:'; command -v sh; echo 'Package manager:'; command -v apk"]
```

Build and run:

```bash
docker build -t task1-alpine:1.0 .
docker run --rm task1-alpine:1.0
```

Compare the image sizes:

```bash
docker image ls task1-ubuntu:1.0 task1-alpine:1.0
```

**Observe and document:**
- The operating-system identification.
- The available package manager (`apt` on Ubuntu and `apk` on Alpine).
- The image sizes shown on your machine.
- Why a container image does not automatically contain every package installed on a full virtual machine.

**Expected learning:** A container image contains its own user-space files and packages, while the running container uses the host machine's kernel.

> Image sizes can change as image tags are updated. Record the sizes you actually observe.

---

# Task 2 — Understand `WORKDIR` and `COPY`

**Goal:** Copy files into an image and understand how source and destination paths work.

## 2.1 Create test files

In the same folder as the Dockerfile, create:

`app.txt`
```text
Hello from app.txt
```

`config.txt`
```text
APP_MODE=demo
```

## 2.2 Dockerfile

```dockerfile
FROM ubuntu:24.04
WORKDIR /app
COPY app.txt .
COPY config.txt .
CMD ["sh", "-c", "echo 'Working directory:'; pwd; echo 'Files:'; ls -l; echo 'app.txt:'; cat app.txt; echo 'config.txt:'; cat config.txt"]
```

Build and run:

```bash
docker build -t task2-copy:1.0 .
docker run --rm task2-copy:1.0
```

## 2.3 Practise a missing-file error

Temporarily change `COPY app.txt .` to:

```dockerfile
COPY missing.txt .
```

Build again:

```bash
docker build -t task2-copy:broken .
```

The build should fail because `missing.txt` is not in the build context. Restore the correct filename and rebuild.

**Observe and document:**
- The meaning of `WORKDIR /app`.
- Why `COPY app.txt .` copies the file to `/app/app.txt`.
- Why the missing source file causes a build error.
- Why files outside the build context cannot normally be copied into the image.

**Expected learning:** `WORKDIR` sets the working directory for subsequent instructions, and `COPY` takes its source from the build context.

---

# Task 3 — `RUN` and the Build Cache

**Goal:** See that `RUN` executes during image building and that build steps may be cached.

## Dockerfile

```dockerfile
FROM ubuntu:24.04
WORKDIR /app
RUN mkdir -p reports
RUN touch reports/build-created.txt
RUN echo "Created during image build" > reports/build-created.txt
CMD ["sh", "-c", "ls -l /app/reports && cat /app/reports/build-created.txt"]
```

Build and run:

```bash
docker build -t task3-run:1.0 .
docker run --rm task3-run:1.0
```

Build it a second time:

```bash
docker build -t task3-run:1.0 .
```

Inspect the output for cached build steps. Now change the text in the last `RUN` command and rebuild:

```bash
docker build -t task3-run:1.1 .
```

**Observe and document:**
- Which build steps are shown as cached.
- Which step runs again after you change it.
- Why `build-created.txt` exists even in a newly created container from the image.
- How this differs from a file created only in a container's writable layer.

**Expected learning:** Filesystem changes made by `RUN` become part of the built image. Docker can reuse cached build steps when the instruction and its dependencies have not changed.

---

# Task 4 — `ARG` vs `ENV`

**Goal:** Demonstrate the difference between build-time variables and runtime environment variables.

## Dockerfile

```dockerfile
FROM ubuntu:24.04

ARG APP_VERSION=1.0
ENV APP_ENV=development

CMD ["sh", "-c", "echo APP_VERSION=${APP_VERSION:-not-set}; echo APP_ENV=$APP_ENV"]
```

Build the default image:

```bash
docker build -t task4-config:1.0 .
```

Run it:

```bash
docker run --rm task4-config:1.0
```

Expected output:

```text
APP_VERSION=not-set
APP_ENV=development
```

Build with a different argument:

```bash
docker build --build-arg APP_VERSION=2.0 -t task4-config:2.0 .
```

Run it:

```bash
docker run --rm task4-config:2.0
```

The `APP_VERSION` output remains `not-set`, because `ARG` is not automatically a runtime environment variable.

Now update the Dockerfile to explicitly transfer the build argument into the image environment:

```dockerfile
FROM ubuntu:24.04

ARG APP_VERSION=1.0
ENV APP_VERSION=${APP_VERSION}
ENV APP_ENV=development

CMD ["sh", "-c", "echo APP_VERSION=$APP_VERSION; echo APP_ENV=$APP_ENV"]
```

Build and run:

```bash
docker build --build-arg APP_VERSION=2.0 -t task4-config:2.1 .
docker run --rm task4-config:2.1
```

Override `APP_ENV` at runtime:

```bash
docker run --rm -e APP_ENV=production task4-config:2.1
```

**Observe and document:**
- How `--build-arg` affects the build.
- Why `ARG` alone is not present in the runtime environment.
- How `ENV` provides a runtime default.
- How `docker run -e` overrides an environment variable for that container.

**Security note:** Do not use `ARG` or `ENV` to store secrets. Build arguments and image environment settings are not appropriate secret-storage mechanisms.

---

# Task 5 — Run as a Non-Root User

**Goal:** Create a user and run the container process with that account.

## Dockerfile

```dockerfile
FROM ubuntu:24.04

RUN useradd --create-home appuser \
    && mkdir -p /app \
    && chown appuser:appuser /app

WORKDIR /app
USER appuser

CMD ["sh", "-c", "echo 'Current user:'; whoami; echo 'User ID:'; id; echo 'Working directory:'; pwd"]
```

Build and run:

```bash
docker build -t task5-user:1.0 .
docker run --rm task5-user:1.0
```

Expected output should identify `appuser` as the current user.

## Permission test

Change the Dockerfile's `CMD` to:

```dockerfile
CMD ["sh", "-c", "touch /root/should-fail.txt"]
```

Build and run:

```bash
docker build -t task5-user:permission-test .
docker run --rm task5-user:permission-test
```

The command should fail because `appuser` normally cannot write to `/root`.

Restore the original `CMD` after the test.

**Observe and document:**
- Why the account must be created before `USER appuser`.
- What `whoami` and `id` show.
- Why writing to `/root` fails.
- Why running an application as a non-root user follows the principle of least privilege.

---

# Task 6 — Master `ENTRYPOINT` and `CMD`

**Goal:** Compare the default command behavior and how runtime arguments are handled.

## 6.1 `CMD` only

```dockerfile
FROM ubuntu:24.04
CMD ["echo", "Hello from CMD"]
```

Build and run:

```bash
docker build -t task6-cmd:1.0 .
docker run --rm task6-cmd:1.0
```

Expected output:

```text
Hello from CMD
```

Run with an override:

```bash
docker run --rm task6-cmd:1.0 echo
```

This replaces the default command and arguments; `echo` runs without the original message.

## 6.2 `ENTRYPOINT` only

Replace the Dockerfile with:

```dockerfile
FROM ubuntu:24.04
ENTRYPOINT ["echo", "Docker"]
```

Build and run:

```bash
docker build -t task6-entrypoint:1.0 .
docker run --rm task6-entrypoint:1.0
docker run --rm task6-entrypoint:1.0 is awesome
```

Expected outputs:

```text
Docker
Docker is awesome
```

## 6.3 Combine `ENTRYPOINT` and `CMD`

```dockerfile
FROM ubuntu:24.04
ENTRYPOINT ["echo"]
CMD ["Hello"]
```

Build and run:

```bash
docker build -t task6-combined:1.0 .
docker run --rm task6-combined:1.0
docker run --rm task6-combined:1.0 Docker
```

Expected outputs:

```text
Hello
Docker
```

**Observe and document:**
- `CMD` supplies a default command and arguments, or default arguments for an `ENTRYPOINT`.
- With exec-form `ENTRYPOINT`, runtime arguments are appended to the entrypoint.
- When `ENTRYPOINT` and `CMD` are combined, runtime arguments replace the default `CMD` arguments.

---

# Task 7 — Understand `EXPOSE` and Port Publishing

**Goal:** Verify that `EXPOSE` documents a port but does not publish it automatically.

This task uses a small HTTP server so the port behavior can be tested directly.

## Dockerfile

```dockerfile
FROM ubuntu:24.04

RUN apt-get update \
    && apt-get install -y --no-install-recommends python3 \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
RUN printf '%s\n' '<h1>Hello from the Docker container</h1>' > index.html

EXPOSE 8080

CMD ["python3", "-m", "http.server", "8080", "--bind", "0.0.0.0"]
```

Build:

```bash
docker build -t task7-web:1.0 .
```

Inspect the image metadata:

```bash
docker image inspect task7-web:1.0 --format '{{json .Config.ExposedPorts}}'
```

Expected metadata includes:

```text
{"8080/tcp":{}}
```

## Run without publishing a host port

```bash
docker run -d --name task7-web-unpublished task7-web:1.0
docker ps --filter name=task7-web-unpublished
docker port task7-web-unpublished
```

No host port mapping is expected. `EXPOSE` alone does not publish the port.

Remove this test container:

```bash
docker rm -f task7-web-unpublished
```

## Publish the port

```bash
docker run -d --name task7-web-published -p 8080:8080 task7-web:1.0
```

Open this URL in a browser:

```text
http://localhost:8080
```

You should see the message `Hello from the Docker container`.

Inspect the mapping:

```bash
docker port task7-web-published
docker logs task7-web-published
```

Clean up:

```bash
docker rm -f task7-web-published
```

**Observe and document:**
- The port metadata created by `EXPOSE`.
- The difference between running without `-p` and with `-p`.
- Why the application must listen on the container port.
- The meaning of `-p HOST_PORT:CONTAINER_PORT`.

This task only tests publishing a container port to the host; it does not cover Docker networking concepts.

---

# Task 8 — Build a Configurable Image

**Goal:** Create one image that reports its version and supports different runtime environments.

## Dockerfile

```dockerfile
FROM ubuntu:24.04

ARG APP_VERSION=1.0
ENV APP_VERSION=${APP_VERSION}
ENV APP_ENV=development

WORKDIR /app

RUN printf '%s\n' '#!/bin/sh' \
    'echo "Application version: $APP_VERSION"' \
    'echo "Application environment: $APP_ENV"' \
    > /app/start.sh \
    && chmod +x /app/start.sh

RUN useradd --create-home appuser \
    && chown -R appuser:appuser /app

USER appuser

EXPOSE 8080

ENTRYPOINT ["/app/start.sh"]
CMD ["default"]
```

Build two versions:

```bash
docker build --build-arg APP_VERSION=1.0 -t task8-app:1.0 .
docker build --build-arg APP_VERSION=2.0 -t task8-app:2.0 .
```

Run the first version:

```bash
docker run --rm task8-app:1.0
```

Override the environment at runtime:

```bash
docker run --rm -e APP_ENV=production task8-app:1.0
```

Run the second version:

```bash
docker run --rm task8-app:2.0
```

Supply a runtime argument:

```bash
docker run --rm task8-app:1.0 custom-argument
```

The script prints the version and environment. The extra argument is passed to the script; this particular script does not use the argument, which is intentional for demonstrating how arguments are passed.

**Observe and document:**
- The version difference between the two images.
- The runtime override for `APP_ENV`.
- The effect of `USER appuser`.
- The role of `ENTRYPOINT` and `CMD`.
- That `EXPOSE` documents the port but this script does not start a server on it.

---

# Task 9 — Dockerfile Troubleshooting Lab

**Goal:** Intentionally create common errors, understand why they occur, and fix them.

Use a separate folder for each test, or restore the Dockerfile between tests.

## Error 1: Invalid base image tag

Dockerfile:

```dockerfile
FROM ubuntu:does-not-exist
CMD ["echo", "hello"]
```

Build:

```bash
docker build -t task9-bad-base:1.0 .
```

**Cause:** Docker cannot find the requested base image tag.

**Fix:** Use a valid tag such as `ubuntu:24.04`.

## Error 2: Missing source file

Dockerfile:

```dockerfile
FROM ubuntu:24.04
WORKDIR /app
COPY missing.txt .
CMD ["ls", "-l"]
```

Build:

```bash
docker build -t task9-bad-copy:1.0 .
```

**Cause:** `missing.txt` is not present in the build context, or it is excluded by `.dockerignore`.

**Fix:** Create the file in the build context or correct the source filename.

## Error 3: User does not exist

Dockerfile:

```dockerfile
FROM ubuntu:24.04
USER appuser
CMD ["whoami"]
```

Build and run:

```bash
docker build -t task9-bad-user:1.0 .
docker run --rm task9-bad-user:1.0
```

**Cause:** The Dockerfile selects `appuser` without creating the account first.

**Fix:**

```dockerfile
FROM ubuntu:24.04
RUN useradd --create-home appuser
USER appuser
CMD ["whoami"]
```

Rebuild and test.

## Error 4: Wrong working directory or command

Dockerfile:

```dockerfile
FROM ubuntu:24.04
WORKDIR /app
CMD ["cat", "missing.txt"]
```

Build and run:

```bash
docker build -t task9-bad-command:1.0 .
docker run --rm task9-bad-command:1.0
```

**Cause:** The command tries to read a file that does not exist in `/app`.

**Fix:** Copy the file into `/app`, create it during the build, or correct the command's filename/path.

## Troubleshooting report template

For each error, record:

| Field | What to write |
|---|---|
| Error | The command or Dockerfile instruction that failed |
| Symptom | The relevant error output |
| Root cause | Why the error occurred |
| Fix | The change made to resolve it |
| Verification | The successful build/run command and result |

**Expected learning:** Read the error carefully, identify whether it happened during build or runtime, and make a targeted correction.

---

# Task 10 — Final Mini Project

**Goal:** Combine the Dockerfile instructions in one configurable, non-root application image.

This mini project uses a simple shell application. It prints configuration values and demonstrates startup behavior. It documents port `8080` but does not start a web server.

## Project structure

```text
docker-mini-project/
├── Dockerfile
├── app.txt
└── README.md
```

Create `app.txt`:

```text
Docker mini project is running.
```

## Dockerfile

```dockerfile
FROM ubuntu:24.04

ARG APP_VERSION=1.0
ENV APP_VERSION=${APP_VERSION}
ENV APP_ENV=development

WORKDIR /app

COPY app.txt .

RUN useradd --create-home appuser \
    && chown -R appuser:appuser /app

USER appuser

EXPOSE 8080

ENTRYPOINT ["sh", "-c", "echo \"Application version: $APP_VERSION\"; echo \"Application environment: $APP_ENV\"; cat /app/app.txt; echo \"Runtime arguments: $*\"", "--"]
CMD ["default"]
```

## Build

From the project directory:

```bash
docker build -t docker-mini-project:1.0 .
```

Build another version:

```bash
docker build --build-arg APP_VERSION=2.0 -t docker-mini-project:2.0 .
```

## Run with defaults

```bash
docker run --rm docker-mini-project:1.0
```

Expected output includes the version, environment, contents of `app.txt`, and the default argument.

## Run in production mode

```bash
docker run --rm -e APP_ENV=production docker-mini-project:1.0
```

## Override the default argument

```bash
docker run --rm docker-mini-project:1.0 demonstration
```

## Verify the configured user

The Dockerfile selects `appuser`. To verify it explicitly, temporarily use this as the `CMD`:

```dockerfile
CMD ["whoami"]
```

Build and run the image again; output should be:

```text
appuser
```

Restore the original `CMD` afterward.

## Mini project checklist

- [ ] `FROM` selects Ubuntu 24.04.
- [ ] `WORKDIR` sets `/app`.
- [ ] `COPY` copies `app.txt` into the image.
- [ ] `RUN` creates the user and adjusts ownership.
- [ ] `ARG` provides a build-time version.
- [ ] `ENV` sets runtime configuration.
- [ ] `EXPOSE` documents port 8080.
- [ ] `USER` selects `appuser`.
- [ ] `ENTRYPOINT` defines the startup executable/behavior.
- [ ] `CMD` supplies a default argument.
- [ ] The image builds and runs successfully.
- [ ] Runtime environment and argument overrides are demonstrated.
- [ ] The project README records build commands, run commands, outputs, and any issues fixed.

---

# Quick Reference: The 10 Instructions

| Instruction | Main purpose | Demonstrated in |
|---|---|---|
| `FROM` | Select the base image | Tasks 1–10 |
| `WORKDIR` | Set the working directory | Tasks 2, 3, 8, 10 |
| `COPY` | Copy files into the image | Tasks 2, 10 |
| `RUN` | Execute commands during image build | Tasks 3, 5, 7, 8, 10 |
| `ARG` | Define a build-time variable | Tasks 4, 8, 10 |
| `ENV` | Set runtime environment defaults | Tasks 4, 8, 10 |
| `EXPOSE` | Document an expected container port | Tasks 7, 8, 10 |
| `USER` | Set the user for subsequent steps/runtime | Tasks 5, 8, 10 |
| `ENTRYPOINT` | Define the main executable | Tasks 6, 8, 10 |
| `CMD` | Set the default command or arguments | Tasks 1–10 |

# Useful Docker Commands

```bash
# List images
docker images

# List running containers
docker ps

# List all containers, including stopped ones
docker ps -a

# View container logs
docker logs CONTAINER_NAME_OR_ID

# Inspect image configuration
docker image inspect IMAGE_NAME

# Remove a stopped container
docker rm CONTAINER_NAME_OR_ID

# Force-remove a container
docker rm -f CONTAINER_NAME_OR_ID

# Remove an image when it is no longer needed
docker image rm IMAGE_NAME
```

## Suggested completion order

1. Finish Tasks 1–3 to understand base images, file placement, and image builds.
2. Finish Tasks 4–5 to practise configuration and least-privilege execution.
3. Finish Tasks 6–8 to understand runtime commands and image configuration.
4. Finish Task 9 to practise troubleshooting.
5. Complete Task 10 and add screenshots/output and explanations to your project README.

**Scope reminder:** This guide intentionally excludes Docker networking concepts and Docker volumes.
