# Running jenkins docker image

based on: https://www.jenkins.io/doc/book/installing/docker/
https://www.jenkins.io/doc/tutorials/build-a-multibranch-pipeline-project/

Create a bridge network in Docker using the following docker network create command:

```bash
docker network create jenkins
```

running basic jenkins container (not required for next steps)
```bash
docker run \
  --name jenkins-docker \
  --rm \
  --detach \
  --privileged \
  --network jenkins \
  --network-alias docker \
  --env DOCKER_TLS_CERTDIR=/certs \
  --volume jenkins-docker-certs:/certs/client \
  --volume jenkins-data:/var/jenkins_home \
  --publish 2376:2376 \
  docker:dind \
  --storage-driver overlay2
```

### Create a custom docker image with  blue-ocean plugin

- create a file named `Dockerfile` with the following content:

```dockerfile
FROM jenkins/jenkins:lts-jdk11
USER root
RUN apt-get update && apt-get install -y lsb-release
RUN curl -fsSLo /usr/share/keyrings/docker-archive-keyring.asc \
  https://download.docker.com/linux/debian/gpg
RUN echo "deb [arch=$(dpkg --print-architecture) \
  signed-by=/usr/share/keyrings/docker-archive-keyring.asc] \
  https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list
RUN apt-get update && apt-get install -y docker-ce-cli
USER jenkins
RUN jenkins-plugin-cli --plugins "blueocean:1.25.5 docker-workflow:1.28"
```

- run the following command from the folder containing the `Dockerfile'

```bash
docker build -t jenkins-blueocean:local .
```

- make shure network is created
```bash
  docker network create jenkins
```
- create persistent volumes
```bash
 docker volume create jenkins-data
 docker volume create jenkins-docker-certs
```

- run the new custom container
```bash
docker run \
  --name jenkins \
  --restart unless-stopped \
  --detach \
  --network jenkins \
  --env DOCKER_HOST=tcp://docker:2376 \
  --env DOCKER_CERT_PATH=/certs/client \
  --env DOCKER_TLS_VERIFY=1 \
  --env JAVA_OPTS=-Dhudson.plugins.git.GitSCM.ALLOW_LOCAL_CHECKOUT=true \
  --publish 8989:8080 \
  --publish 50000:50000 \
  --volume jenkins-data:/var/jenkins_home \
  --volume jenkins-docker-certs:/certs/client:ro \
  --volume "$HOME":/home \
  jenkins-blueocean:local
```

Open jenkins in the browser:
http://localhost:8989

- unlock jenkins using the automatically-generated password

```bash
docker exec jenkins-blueocean cat /var/jenkins_home/secrets/initialAdminPassword
```

- setup admin account
- install recommended plugins

Not really needed as we use persistent volumes:
create new image with the latest state, otherwise state is lost when stopping the container,
```bash
docker commit [CONTAINER_ID] [new_image_name]
```
list images
```bash
docker images
```

some useful commands
```bash
# get commandline
docker exec -it jenkins-blueocean bash
# tail log
docker logs <docker-container-name>
```
```bash
docker container exec -it <docker-container-name> bash  
docker container ls 
```

# Setup local git repo

```bash
#Create remote repo
git init --bare ~/repo/remote/docker_jenkins_pipelines.git

#Create locale repo
git init docker_jenkins_pipelines
cd docker_jenkins_pipelines
touch .gitignore
git status
git add .
git commit -m "initial commit"
# Connect to remote repo
git remote add origin ~/repo/remote/docker_jenkins_pipelines.git
git push --set-upstream origin master
# set remote url (not needed)
git remote set-url origin ~/repo/remote/docker_jenkins_pipelines.git
```

# allow local git repo, (this seems not reliable, easier to set from docker run as used above)
From Manage Jenkins -> script console:
hudson.plugins.git.GitSCM.ALLOW_LOCAL_CHECKOUT=true
```groovy
System.setProperty("hudson.plugins.git.GitSCM.ALLOW_LOCAL_CHECKOUT", "true")
// and validate
System.getProperty("hudson.plugins.git.GitSCM.ALLOW_LOCAL_CHECKOUT")
```

# Create the pipeline from this repo
In jenkins at "new item" -> select multi pipeline and use this
url for this repo: `/home/repo/remote/docker_jenkins_pipelines.git`