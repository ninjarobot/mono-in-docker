Mono Applications in Docker Containers
======================================

Mono applications can be run in docker containers in several ways, including using base images from Xamarin or Microsoft that contain mono.  If you have some specific runtime requirements, you may need to build your own images with mono preinstalled or provided on a data volume when launching the container.

Using mono from host
-----------------------

Because mono applications are often very self-contained, you can create a very slim image and make the mono runtime from the host available to the container when launching.  This is convenient for keeping all applications using the same version of the mono runtime, allowing you to upgrade the runtime without modifying the existing images.

Dockerfile
```
from centos
ENV PATH /mnt/mono/bin:$PATH
ENV LD_LIBRARY_PATH=/mnt/mono/lib:$LD_LIBRARY_PATH
```

This short Dockerfile expects the mono runtime will be provided as a data volumefrom the host under the /mnt/mono mount point.  

Build with `docker build -t external_mono .` to get a new docker image based on CentOS named "external_mono".  This image doesn't contain mono, and instead you should pass mono in as a data volume from the host.

```
docker run -v /opt/mono:/mnt/mono --rm external_mono mono --version
```

The command above runs `mono --version` using the mono that is provided by the host.  The host makes `/opt/mono` available as a data volume at `/mnt/mono`.  The `--rm` option cleans up the container on exit. 

As you'll usually need an application there as well, you can similarly pass the application to the container as another data volume.

```
docker run -v /opt/mono:/mnt/mono -v /var/lib/myapp:/mnt/myapp --rm external_mono mono /mnt/myapp/MyExecutable.exe
```

With this approach, you never really build new images, which is very useful for simply sandboxing applications.  The drawback is that all containers are running the same version of mono, shared from the host.  If you run an orchestration system, such as Apache Mesos, the runtime must be installed on all hosts (Mesos slaves in this case).

Including mono in image
-----------------------
When all the dependencies are included within the image, the docker containers are much easier to move around because there are no additional host dependencies.  This also means that containers can run different versions of the mono runtime if needed.  The added flexibility comes at a small cost - images contain the runtime, so they are larger.

Dockerfile
```
from centos
RUN rpm --import "http://keyserver.ubuntu.com/pks/lookup?op=get&search=0x3FA7E0328081BFF6A14DA29AA6A19B38D3D831EF"
RUN yum-config-manager --add-repo http://download.mono-project.com/repo/centos/
RUN yum -y install mono-complete
```

The image is larger, but nothing is required on the host, and now you can run mono directly in the container without a data volume:

```
docker run --rm included_mono mono --version
```

This image can be easily reused with any mono application that need to run on CentOS by attaching a data volume containing the application and then running it:

```
docker run -v /var/lib/myapp:/mnt/myapp --rm included_mono mono /mnt/myapp/MyExecutable.exe
```

The next step is to build an application-specific image on top of this image.

Dockerfile
```
from included_mono
COPY MyExecutable.exe /var/lib/MyApplication/
```

Building with `docker build -t mono_with_app .` will build an image that contains the application, and you can run it with no data volumes needed:

```
docker run --rm mono-with-app mono /var/lib/MyApplication/MyExecutable.exe
```

This option is definitely the simplest to run, and provides you with a base image for building additional images.  Each image takes up space, however.  Images with base CentOS and no mono are ~200 MB, whereas images with mono included are over 500 MB.
