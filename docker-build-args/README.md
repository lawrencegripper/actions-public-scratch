# Docker Build Args Example

This folder contains Dockerfiles and a GitHub Actions workflow to demonstrate building Docker images with user mapping and running containers with different user permissions.

It highlights how issues can occur when:
1. `useradd` is used to create a user during docker build with UID=1000 and then `docker run` is used with that image to write to `$GITHUB_OUTPUT` 
2. `COPY --chmod=660` is used during build with UID=1000 and then `--user` is used during run with UID=1001
3. `docker run --user` is used by the `docker build` was run with a different user

## Files

- [./.github/docker-build-args.yml](../.github/workflows/docker-build-args.yml)
- Docker files in [.docker-build-args](./)
    - `Dockerfile.with-user-mapping`: Builds a Docker image with user and group IDs passed as build arguments.
    - `Dockerfile.without-user-mapping`: Builds a Docker image without user and group ID mapping.
    - `Dockerfile.with-sudo`: Builds a Docker image with sudo installed to allow running commands with elevated privileges.

The workflow demonstrates different approaches to mapping users between the host and the container.

It demonstrates how these work or fail in different configurations, both with files mounted and files copied in at build time.

### File mounted into the container at run time

There are two of these 
1. [file-mounted-from-host.txt](./file-mounted-from-host.txt) which is mounted like so `-v ${{ github.workspace }}/docker-build-args:/docker-build-args` 
2. `$GITHUB_OUTPUT` which is mounted like so `-v $GITHUB_OUTPUT:$GITHUB_OUTPUT` and used to set stage outputs

### File copied into the container a build time

In the docker file we run:

```
COPY --chmod=660 --chown exampleusername:examplegroupname ./docker-build-args/660-user-owned-copied-into-docker-image.txt .
```

This simulates creating a file which is owned and readable only by the docker files current `USER`.

In a real example these could be created by doing `RUN npm install` which pulls and places files in the image.

## GitHub Actions Workflow Examples and Outcomes

The workflow file `.github/workflows/docker-build-args.yml` defines several jobs to build and run Docker images with different user permissions and we can see the outcomes.

- [Resulting workflow](https://github.com/lawrencegripper/actions-public-scratch/actions/runs/12430924833/job/34707316182)

### Job: `Build with user mapping, run as container user`

Uses `docker build --args` to build image with UID that matches current runner UID.

https://github.com/lawrencegripper/actions-public-scratch/blob/ad79417943183c87e0efa72f8d6c2ea3da3950d8/docker-build-args/Dockerfile.with-user-mapping#L3-L14

https://github.com/lawrencegripper/actions-public-scratch/blob/ad79417943183c87e0efa72f8d6c2ea3da3950d8/.github/workflows/docker-build-args.yml#L58-L62

**Will pass?** ✅

Can `docker run`:
 - write to file placed in built image `/app/660-user-owned-copied-into-docker-image.txt` ✅
 - write to file mounted from build folder `/docker-build-args/file-mounted-from-host.txt` ✅
 - write to $GITHUB_OUTPUT ✅

### Job `Build with user mapping, run with user mapping`

[Adds](https://github.com/lawrencegripper/actions-public-scratch/blob/ad79417943183c87e0efa72f8d6c2ea3da3950d8/.github/workflows/docker-build-args.yml#L112) `--user "$(id -u):$(id -g)"` to the `docker run` command.

Same as 👆, because the `docker build` create as maps the user the `docker run --user` arg is 
redundant. The user is already configured to match current UID `1001`.

### Job: Build without user mapping (UID=1000), run with user mapping

**Will pass?** ❌

Can `docker run`:
 - write to file placed in built image `/app/660-user-owned-copied-into-docker-image.txt` ❌ [Example](https://github.com/lawrencegripper/actions-public-scratch/actions/runs/12430732525/job/34706736062#step:5:48)
 - write to $GITHUB_OUTPUT ✅

#### Why?

The `docker run` has `UID=1001`. As the UID was not mapped during build the `/app/660-user-owned-copied-into-docker-image.txt` file is only writable to `UID=1000`.

It can still write to `$GITHUB_OUTPUT` because this file is writable by `UID=1001`.

### Job: `Build without mapping, run with without mapping`

**Will pass?** ❌

Can `docker run`:
 - write to file placed in built image `/app/660-user-owned-copied-into-docker-image.txt` ✅
 - write to file mounted from build folder `/docker-build-args/file-mounted-from-host.txt` ❌ [Example](https://github.com/lawrencegripper/actions-public-scratch/actions/runs/12430924833/job/34707316191#step:5:51)
 - write to $GITHUB_OUTPUT ❌ [Example](https://github.com/lawrencegripper/actions-public-scratch/actions/runs/12430924833/job/34707316191#step:6:41)

#### Why?

Writing to file added during build is fine as the user for `docker run` is the same. 

Writing to mounted file `/docker-build-args/file-mounted-from-host.txt: Permission denied` as the file is owned by `UID=1001` but docker container user is `UID=1000` 

Writing to $GITHUB_OUTPUT fails for the same reason.

### Job: `Build without mapping, run without user mapping, use sudo to write to GITHUB_OUTPUT`

**Will pass?** ✅

Can `docker run`:
 - write to file placed in built image `/app/660-user-owned-copied-into-docker-image.txt` ✅
 - write to file mounted from build folder `/docker-build-args/file-mounted-from-host.txt` ✅
 - write to $GITHUB_OUTPUT ✅

#### Why?

The container has `sudo` installed. When using `docker run` we use `sudo` to make the user who runs 
the command `root` `UID=0` giving it access to all files.

### Job: `No user mapping, build and run as root user`

**Will pass?** ✅

Can `docker run`:
 - write to file placed in built image `/app/660-user-owned-copied-into-docker-image.txt` ✅
 - write to file mounted from build folder `/docker-build-args/file-mounted-from-host.txt` ✅
 - write to $GITHUB_OUTPUT ✅

#### Why?

The user in the container is `root` `UID=0` this gives it write permissions over all files regardless of owning user.
