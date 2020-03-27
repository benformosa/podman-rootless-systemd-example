# Podman rootless and systemd example service

Deploy a Docker Compose application with rootless Podman as a systemd service.

This example deploys the [Example Voting App](https://github.com/dockersamples/example-voting-app).

## Goals

This project demonstrates deploying a Docker Compose application using the Red Hat-supported container engine podman, using systemd to manage it as a service.

* Deploy application from Docker Compose file with rootless Podman as the container engine
* Install application as a systemd service
* Support HTTP proxy for downloading container images
* Do not require Internet access to run the application
* Do not require root to run the application

## Requirements

* Red Hat Enterprise Linux 7.7
* Python 3.6 from Red Hat Software Collections. See [Red Hat Developer Blog - How to install Python 3 on Red Hat Enterprise Linux](https://developers.redhat.com/blog/2018/08/13/install-python3-rhel/)
* [podman-compose](https://github.com/containers/podman-compose)
* pyyaml

## Quick start

```bash
scl enable rh-python36 "python -m venv ./venv"
source venv/bin/activate
pip install --require-hashes -r requirements.txt
podman-compose up -d
```

Vote at http://localhost:5000/

View results at http://localhost:5001/

## Configure systemd service

Configure the application as a systemd service.

You will need super-user rights, but the application will run as a new, unprivileged user `eva`.

1. Ensure that the system is updated to RHEL 7.7.
    ```console
    $ cat /etc/redhat-release
    Red Hat Enterprise Linux Server release 7.7 (Maipo)
    ```
1. Create a user to run the service.
    ```bash
    sudo useradd eva
    ```
1. Ensure that the user has about 12 GiB disk space free in their home directory.
    ```bash
    df -h /home/eva
    ```
1. Install the service unit file to the local configuration.
    ```bash
    cp example-voting-app.service /etc/systemd/system/
    ```
1. Modify `example-voting-app.service` and set the user, group and path to application directory. The absolute path must be specified.  
Make sure to update each instance of `User`, `Group`, `WorkingDirectory`, `ExecStartPre`, `ExecStart`, `ExecStop`.
    ```bash
    sudo vi /etc/systemd/system/example-voting-app.service
    ```
1. Install pre-requisite software.
    ```bash
    sudo yum install podman skopeo rh-python36
    ```
1. Create a directory to store Python modules.
    ```bash
    sudo mkdir -p /opt/eva-python/packages
    ```
1. Download packages and move into store.  
pip supports the `HTTPS_PROXY` environment variable.
    ```bash
    mkdir /tmp/packages
    scl enable rh-python36 "pip download --require-hashes -r requirements.txt --dest /tmp/packages"
    sudo cp /tmp/packages/* /opt/eva-python/packages/
    rm -r /tmp/packages
    ```
1. Pull images from the Internet and copy into the user's container store.  
Ensure that `/tmp` has at least 600 MiB disk space free.  
skopeo supports the  `HTTPS_PROXY` environment variable.
    ```bash
    for image in $(echo $(grep image docker-compose.yml | sed 's/^.*: //') "k8s.gcr.io/pause:3.1"); do
        image_name=$(echo $image | cut -d: -f1)
        mkdir -p /tmp/images/${image_name}
        skopeo copy docker://${image} oci:/tmp/images/${image}
        sudo -u eva -i skopeo copy oci:/tmp/images/${image} containers-storage:${image}
        rm -r /tmp/images/${image_name}
    done
    rm -r /tmp/images
    ```
1. Enable and start the service.
    ```bash
    sudo systemctl enable --now example-voting-app
    ```

## Changes to the Docker Compose file.

In order to work correctly with Podman Compose, and to avoid needing to build images locally, some changes were required to the Docker Compose file.

[docker-compose.yml](docker-compose.yml) is a modified version of [docker-compose-simple.yml](https://github.com/dockersamples/example-voting-app/blob/master/docker-compose-simple.yml) from the [dockersamples/example-voting-app](https://github.com/dockersamples/example-voting-app/) repository.

### Changes

* Replace the `build` configuration options with `image`, to pull pre-built images from [Docker Hub](https://docker.io).
* Remove the `mount` configuration options, which are designed for use with a local copy of the original repo. See dockersamples/example-voting-app#153
* Change the port used by the vote service. Podman Compose deploys all containers into a single pod, so using the same container port more than once won't work. Thankfully, this is simple to do with a `command` option, although specifying this option as a string, rather than a list, did not work.
* Add volume mounts to the `db` and `redis` services. Podman will always create volumes for these containers anyway, because they are declared in the image.
    * Setting a volume mount will allow the data to persist if the containers are removed.
    * Be careful not to run `podman volume prune`, as the volume's path is used by the container, but from Podman's perspective, it is not used by the container.
    * Without this change, Podman Compose will use Podman's default action for image volumes to create and bind mount a new volume. See the `--image-volume` option in `man 1 podman-create`.

## License

Apache-2.0

## Author Information

Ben Formosa

© Commonwealth of Australia, 2020
