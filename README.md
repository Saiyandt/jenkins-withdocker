#  The simple way to run Docker-in-Docker for CI

Docker doesn't recommend running the Docker daemon inside a container (except for very few use cases like developing Docker itself), and the solutions to make this happen are generally hacky and/or unreliable.

However, there is an easy workaround: mount the host machine's Docker socket in the container. This will allow your container to use the host machine's Docker daemon to run containers and build images.

Your container still needs compatible Docker client binaries in it, but I have found this to be acceptable for all my use cases.

The version using my prebuilt image (the docker build is in this url: ):
```
    docker run \
     -p 8080:8080 \
     -v /var/run/docker.sock:/var/run/docker.sock \
     --name jenkins \
     jenkins-withdocker:1
```
The guide below is for Jenkins, but you can apply the same logic to any other build server.

Building Docker containers with Jenkins inside a container
First, we'll run Jenkins as a container using the official Jenkins image:
```
    docker run -p 8080:8080 \
      -v /var/run/docker.sock:/var/run/docker.sock \
      --name jenkins \
      jenkins/jenkins:lts
```
Note that the key here is mounting /var/run/docker.sock from the host machine to the same location inside the container.

Then, we'll need to install the Docker binaries inside the container. Spawn an interactive shell inside the running Jenkins container:
```
	docker exec -it -u root jenkins bash
```
Because the official Jenkins image is based on Debian 9, we can use apt to install the Docker binaries as instructed in the Docker installation guide. This is a single snippet to install some prerequisites, configure the official Docker apt repositories and install the latest Docker CE binaries:
```
    apt-get update && \
    apt-get -y install apt-transport-https \
         ca-certificates \
         curl \
         gnupg2 \
         software-properties-common && \
    curl -fsSL https://download.docker.com/linux/$(. /etc/os-release; echo "$ID")/gpg > /tmp/dkey; apt-key add /tmp/dkey && \
    add-apt-repository \
       "deb [arch=amd64] https://download.docker.com/linux/$(. /etc/os-release; echo "$ID") \
       $(lsb_release -cs) \
       stable" && \
    apt-get update && \
    apt-get -y install docker-ce
```
Note: The Docker daemon running on your host machine must be compatible with the version of client binaries you are installing. To verify the version, run docker version on your host machine.

-> Your Jenkins container should now have a functioning Docker installation. Verify by running:

    docker ps

The output should be the same as when running the command on the host machine (you should at least expect to see the Jenkins container running!).

Next, you'll need to finish the installation of Jenkins as usual. Open the Jenkins installer by navigating to http://localhost:8080.

You will need the initial admin password, which can be obtained by running:

    docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword

Next select "Install recommended plugins", and wait for Jenkins to install everything. The Docker Pipeline plugin is installed by default, which means you are ready to go!
