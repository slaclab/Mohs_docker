from: https://gitlab.lbl.gov/ci-cd/mohs-basic

Docker container based testing
==============================

Recipes and Docker images to support LBL CI/CD servers.

### Introduction

The support for doing CI through running tests on a bare machine in a shell
eventually gets painful because of ever changing dependencies and their versions,
for this single reason it is better to move to a more abstract system like docker.

#### Advantages

1. If you develop gateware/firmware and need motivation look at
    this [talk](https://ohwr.org/project/ohr-meta/wikis/uploads/bf22cc74d57b32e19f26df532ad80c02/ci_tests_talk.pdf)
2. Track dependencies and requirements through Dockerfile
3. Ease of use with Docker (Industry seems to be running with it, and a lot of
    work to leverage upon)
4. Most CI out there is currently being done with docker
5. When changes are made to Dockerfile, the updated dependencies get installed
    automatically.

#### Disadvantages

1. Not so lean. Linux images are big, especially, when stuff needs to be
    installed by hand, and there are no two ways about it.
2. Managing requires a bit of investment into the docker ecosystem, which isn't
    small.

### Using Docker in CI/CD

We are using Docker as the basis of our CI/CD strategy>

#### References

1. [A good introduction to Docker](https://docs.docker.com/engine/docker-overview/)
2. There seem to be obvious
    [pitfalls in terms of security](https://docs.docker.com/engine/security/security/#docker-daemon-attack-surface)

#### Key takeaways

What I take from here is that:

1. Don't expose the https API (duh?)
2. Don't run anything else on the server explicitly other than Docker. Not sure
what this means for having a gitlab-runner. Actually, I do, see 3.
3. Looks like it makes sense to run gitlab-runner itself inside a docker
    container as mentioned [here](https://docs.gitlab.com/runner/install/docker.html).
    This will allow having a single software with sudo privileges (i.e. Docker)
4. Overall, still not happy to give docker root privileges. A lot of work
    can/needs to be done using USER
5. The way I have overcome this is by not exposing the machine to outside world
    outside the trusted ports 80, 22 etc. And running the docker registry
    (advanced, see below) locally only.

### Instructions on how to reproduce LBL CI/CD architecture

1. Install docker

Follow instructions from [docker.com](https://docs.docker.com/install/linux/docker-ce/debian/)
to install Docker CE stable.Nothing interesting really, once you trust
docker.com (and its supply chain) with root on your machine.

2. Run gitlab runner inside a docker container

This uses the latest gitlab-runner container maintained on docker hub by gitlab.

```bash
docker run -d --name gitlab-runner --restart always -v /srv/gitlab-runner/config:/etc/gitlab-runner -v /var/run/docker.sock:/var/run/docker.sock    gitlab/gitlab-runner:latest
```

3. Strategy of CI with Docker

Want to run tests under a certain reproducible configuration?

    1. Write a Dockerfile for your setup
    2. Build docker image from the file.
    3. Add it to your repo for tracking
    4. Load them to the CI server


     ```bash
    docker save my-awesome-image | ssh my-docker-running-machine "docker load"
    ```

    5. Update .gitlab-ci.yml to use this image to run tests (see bedrock/.gitlab-ci.yml)
    6. For advanced registry setup see below

4. Register the gitlab-runner with gitlab server

``` bash
docker run --rm -t -i -v /srv/gitlab-runner/config:/etc/gitlab-runner gitlab/gitlab-runner register --non-interactive --executor "docker" --docker-image iverilog-test:latest --url "https://gitlab.lbl.gov/" --registration-token "<REG_TOKEN>" --description "my-docker-runner" --run-untagged --locked="false"
```
This should successfully register the container and updates /srv/gitlab-runner/config/config.toml.
Have a look at it.

If you have a non-standard git hosting service (unlike gitlab.com or github.com),
your self-hosted gitlab server may put you in an awkward position with a few things.

5. Certs for https are self-certified, so you would need to add them to the gitlab-runner container

```bash
scp -pr gitlab.lbl.gov.{pem,crt} mohs:/srv/gitlab-runner/config/certs/.
```

As shown [here](https://gitlab.com/gitlab-org/gitlab-runner/issues/3748), copy
gitlab.lbl.gov.crt to /srv/gitlab-runner/config/certs/.

6. Once the runner inside the container is registered. The docker executor is
set to execute:

    1. Create cache container to store all volumes as defined in config.toml and Dockerfile of build image (my-awesome-image:latest)
    2. Create build container and link any service container to build container.
    3. Start build container and send job script to the container.
    4. Run job script.
    5. Checkout your latest commit from your git-repo in: /builds/group-name/project-name/.
    6. Run any steps you may have defined in .gitlab-ci.yml.
    7. Check exit status of build script, and report it as FAIL or SUCCESS
    8. Remove build container and all created service containers.

    Now if your certificates aren't "legit", for checking out the code you
    should set GIT to not verify SSL, as I couldn't get this to work in any other way.
    `GIT_SSL_NO_VERIFY` to false as shown [here](https://gitlab.com/gitlab-org/gitlab-runner/issues/986)

### Docker tips

1. In case the rootfs is starting to get full, you can move the `/var/lib/docker` into
a separate directory. References: [here](https://stackoverflow.com/questions/40755494/docker-compose-internal-error-cannot-create-temporary-directory), [here](https://forums.docker.com/t/how-do-i-change-the-docker-image-installation-directory/1169)
and [here](https://linuxconfig.org/how-to-move-docker-s-default-var-lib-docker-to-another-directory-on-ubuntu-debian-linux)

2. Removing containers and images in case of bloat

```bash
# Remove all stopped containers
docker rm $(docker ps -a -q)
# Remove all containers
docker rm -f $(docker ps -a -q)
# Remove all images
docker rmi $(docker images -q)
```

Reference [here](https://stackoverflow.com/questions/40755494/docker-compose-internal-error-cannot-create-temporary-directory)

3. More useful commands

```bash
# This lists all images
docker image ls --all
# This removes an image
docker rmi <image name>
# This lists all containers
docker container ls --all
# Lists all docker processes
docker ps --all
# Run a shell attached to the container
docker run --rm -i -t my-awesome-image /bin/bash -c 'ls'
```

4. Useful cheat sheet

Docker cheat sheet [here](https://www.docker.com/sites/default/files/Docker_CheatSheet_08.09.2016_0.pdf)

5. Mounting a device inside docker container

* You can do this automatically per runner basis from /srv/gitlab-runner/config/config.toml
* Per docker runner you can mount devices with `--device` flag.
* Here is an example way to pass usb subsystem into the container.

```bash
docker run --rm -it --device=/dev/ttyUSB0:/dev/ttyUSB0 mohs.dhcp.lbl.gov/testing_base /bin/bash -c "miniterm.py"
```

6. Effects of migrating CI to github. Check [this repository](https://github.com/ligurio/awesome-ci)

7. Nothing out there that is truly free and steady to use, outside self hosted
gitlab (which has some features missing like integrating with external repos).

May have to dish out money to TravisCI or CircleCI or even Gitlab Premium
(which isn't hosted by us).Looks like Jenkins integration to Gitlab has just
opened up and looks promising.

8. Automatically building all required Docker images

    1. As Dockerfiles are revision controlled. Upon a change to the them, the
        images are rebuilt.
    2. If there is no image cache they are completely rebuilt. And if your
        requirements are a lot, then it adds time in addition to building your
        Vivado bitfiles. Especially if you are building things like
       riscv-toolchain from source :)
    3. These rebuilds can be cached in a local registry or a place like
        dockerhub. If you are not the one to be turned-off by running your
        own registry, you have company.
    4. Between changes to dependencies the images are cached in the registry,
        and containers are spawned instantaneously. The new images can be set
        to be rebuilt on per-git-branch basis so each development branch can
        have the luxury of its own dependencies.
    5. This rebuild is in itself done through CI, so a docker engine, running
        inside another docker engine. This comes with its own good and bad. And
        information is over [here](https://docs.gitlab.com/ee/ci/docker/using_docker_build.html).
        This is also the recommended way. See [bedrock/.gitlab-ci.yml] for how
        this is done.
    6. Overall, it is important to watch out who is able to commit to this build
        repo. Because this Docker inside Docker runs as privileged container, it
        has privileges to modify the parent docker, and that could potentially delete
        all the containers?!
    7. These rebuilt images are then pushed to a local registry with a version
        tag `latest`

### Setting up a docker registry

Basic steps:

1. Run a docker registry
2. Automatically build Dockerfiles from mohs-basic through another CI instance
and send them to the registry
3. Update .gitlab-ci.yml to use images from the registry
4. Local registry setup with self-signed certificates:

```bash
# Create a certificate: enter mohs.dhcp.lbl.gov for "Common Name", everything else can be gibberish
openssl req -newkey rsa:4096 -nodes -sha256 -keyout certs/domain.key -x509 -days 365 -out certs/domain.crt
sudo cp certs/domain.crt /etc/docker/certs.d/mohs.dhcp.lbl.gov/ca.crt

# Add the self signed certificate to a list of docker certs so that the cert is recognized by docker as it queries the registry
# After this step the "local docker machine" should be able to make a query to the registry

# Below adds .crt to system (mohs's) certificates and updates the system
sudo cp certs/domain.crt /usr/local/share/ca-certificates/mohs.dhcp.lbl.gov.crt
sudo update-ca-certificates
```

### Contribute

* This is an evolving document, please feel free to contribute
* Rigor is lacking towards gitlab registry end, as I am not aware of any readership,
    once I am convinced this is being read, I would be happy to put in more detail
