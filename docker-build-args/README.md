# Docker Build Args Example

This folder contains Dockerfiles and a GitHub Actions workflow to demonstrate building Docker images with user mapping and running containers with different user permissions.

It highlights how issues can occur when 
1. `useradd` is used to create a user during docker build with UID=1000 and then `docker run` is used with that image to write to `$GITHUB_OUTPUT` 
2. `COPY --chmod=660` is used during build with UID=1000 and then `--user` is used during run with UID=1001

## Dockerfiles

- `Dockerfile.with-user-mapping`: Builds a Docker image with user and group IDs passed as build arguments.
- `Dockerfile.without-user-mapping`: Builds a Docker image without user and group ID mapping.
- `Dockerfile.with-sudo`: Builds a Docker image with sudo installed to allow running commands with elevated privileges.

In the docker file we run:

```
COPY --chmod=660 --chown exampleusername:examplegroupname ./docker-build-args/660-user-owned-copied-into-docker-image.txt .
```

This simulates creating a file which is owned and readable only by the docker files current `USER`. 

In a real example these could be created by doing `RUN npm install` which pulls and places files in the image.

## GitHub Actions Workflow

The workflow file `.github/workflows/docker-build-args.yml` defines several jobs to build and run Docker images with different user permissions:

- **build-with-user-mapping**: Builds and runs a Docker image with user mapping.
- **build-with-user-mapping-run-with-user-mapping**: Builds and runs a Docker image with user mapping, explicitly setting the user during `docker run`.
    - This is redundant as the image was build with user ID mapped by default `docker run` will use that user
- **build-without-user-mapping-run-with-user-mapping**: Builds a Docker image without user mapping and runs it with user mapping.
    - Will ❌ 
    - The `./app/660-user-owned-copied-into-docker-image.txt` copied into the container at build time is owned by `1001` but the running `docker run --user` with `1001` means it cannot write to that file.
- **build-as-1000-run-without-mapping**: Builds a Docker image as user 1000 and runs it without user mapping.

- **build-as-1000-run-with-user-mapping**: Builds a Docker image as user 1000 and runs it with user mapping.
    - Will ❌ [Example](https://github.com/lawrencegripper/actions-public-scratch/actions/runs/12430561238/job/34706233767#step:6:37)
    - The user in the container `1000` can't write to the $GITHUB_OUTPUT as this is only writable to user `1001`
- **build-as-1000-run-with-user-mapping-with-sudo**: Builds a Docker image as user 1000, runs it with user mapping, and uses sudo to write to `GITHUB_OUTPUT`.
- **no-user-mapping**: Builds and runs a Docker image as the root user.

Each job includes steps to check out the repository, build the Docker image, and run the container with various commands to check file permissions and user information.

## Environment Variables

The workflow uses several environment variables to define commands for checking permissions and writing to files:

- `USER_PERMISSIONS_OUTPUT_CMD`: Outputs user and group information and lists directory permissions.
- `WRITE_PERMISSION_CHECK_CMD`: Checks write permissions for files added to the Docker image and files mounted from the host.
- `OUTPUT_PERMISSION_CHECK_CMD`: Attempts to set output for the workflow stage.

These commands are executed inside the Docker containers to verify the behavior of different user mappings and permissions.