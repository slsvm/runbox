# Runbox: Persistent Elastic Execution Environment on Kubernetes

## Prerequisites
Tested configuration:
  - `minikube` with --vm-driver=none (can also use a GKE cluster - see at the bottom)
  - `docker` (client)
  - `kubectl`
  - `python` (`$ sudo apt-get install python3-pip`)
  - `unison` (`$ sudo apt-get install unison`) - for filesystem synchronization
  - `docker-squash` (modified)

     - NOTE: the respective `runbox` command (`merge`) currently does not work, so you can skip this dependency too

    ```
    $ pip3 install --upgrade https://github.com/glikson/docker-squash/archive/commit_message.zip
    ```

## Usage

### Initialization
 - Configure local `docker` to work with minikube's engine:

    ```
    $ minikube docker-env
    # export the specified DOCKER variables
    ```
 - Build `rbutil` image on minikube's Docker engine:

    ```
    $ docker build -t rbutil rbutil/
    ```

### Creating environment

```
$ bin/runbox --create rb1

$ bin/runbox rb1 hostname
rb1
```

User can specify Docker image (using 'ubuntu:18:10' by default):

```
$ bin/runbox --create rb2 --image picoded/ubuntu-openjdk-8-jdk

$ bin/runbox rb2 javac -version
javac 1.8.0_191
```

### Recycling Pods

Underlying Kubernetes Pods can be removed when idle. This can be done explicitly, by adding `--kill_after` flag (e.g.: `bin/runbox --kill_after rb1 true`) - or via a policy enforced by Runbox Controller (yet to be implemented).

### Persistent image
For every environment, Runbox maintains a tagged Docker image (with the same 
name - e.g., 'rb1:latest'). Every time a Pod is recycled (e.g., using
 `--kill_after`), a snapshot of the image is taken and is tagged with the same
tag. This way next time the Pod is restarted, the container will see the same
state of the image.

```
$ bin/runbox rb1 pwd
/

$ bin/runbox --kill_after rb1 touch aa

$ bin/runbox rb1 ls aa
aa
```

### Accessing persistent volume

Persistent volume is mounted at /data

```
$ bin/runbox rb1 touch /data/rb1file

$ bin/runbox rb1 ls /data
rb1file

```

Synchronizing the `data` folder with the persistent volume in the environment
```
$ ls ./data
README.md
$ bin/runbox --sync_before rb1 ls /data
README.md rb1file

$ ls ./data
README.md rb1file
$ touch ./data/localfile
$ bin/runbox --sync_before rb1 ls /data
README.md localfile rb1file

```

Currently, volumes are implemented using NFS, with folders created for each
environment (nfs-server Kubernetes service is deployed automatically, if not
deployed already).


### Adjusting resource allocation
By default, a new environment is configured to require 1/8 CPU core and 1/8 GB
of memory. It is possible to change the allocation, by ussing `--allocate` flag
when running a command. The argument specifies the multiplier to be applied to
the default CPU and memory allocation (e.g., if `2` is specified, container
will be allocated 1/4 core and 1/4 GB memory).The new allocation persists to
future command executions.
```
$ runbox -c rb1

$ runbox rb1 cat /sys/fs/cgroup/memory/memory.limit_in_bytes
134217728

$ runbox --allocate 0.5 --kill_after rb1 cat /sys/fs/cgroup/memory/memory.limit_in_bytes
67108864

$ runbox rb1 cat /sys/fs/cgroup/memory/memory.limit_in_bytes
67108864
```

## Using Runbox with GKE
In order to use runboxes on GKE:

 - Set RBPLATFORM environment variable to `GKE`:

    ```
    $ export RBPLATFORM=GKE
    ```
 - Verify your `gcloud` client is configured to the project you want to use (NOTE: this is important, runbox uses the same command internally):

    ```
    $ export PROJECT_ID=`gcloud config list 2>&1 | grep -Po 'project = \K.*'`
    $ echo $PROJECT_ID
    ```
 - Authenticate local `kubectl` with GKE

    ```
    $ gcloud container clusters get-credentials slsvm-cluster --zone us-central1-a --project ${PROJECT_ID}
    ```
 - Authenticate local `docker` with GCR

    ```
    $ docker login -u oauth2accesstoken -p $(gcloud auth print-access-token) https://gcr.io
    ```
 - Push `rbutil` image to GCR:

    ```
    $ docker build -t gcr.io/${PROJECT_ID}/rbutil rbutil/
    $ docker push gcr.io/${PROJECT_ID}/rbutil
    ```
 - Verify that it works (the first time it will take few seconds)

    ```
    $ runbox --create gkerb1 --image alpine
    $ runbox gkerb1 hostname
    gkerb1
    ```
