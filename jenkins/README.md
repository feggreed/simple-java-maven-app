Overview
Jenkins can be easily run inside the Docker container. There are two types of persistent data that you do not want to lose here:
•	Jenkins configuration (includes tasks)
•	task artifacts or build results.
Usually developers/devops take care about build results, from other hand they often forget about CI system configuration. Jenkins team has introduced a pipeline mechanism that allows developer to store a task configuration directly in the application code (using Jenkinsfile). This helps to reduce the number of things that are stored on Jenkins side. Allows change build configs directly in the code, as plus you get configuration versioning.

What to do with the remaining Jenkins configuration? One of ways is to store it in the SCM repository. In this case we just need to start a pre-configured Jenkins container on a new setup, Jenkins pulls configuration from repository, and then becomes ready to run its tasks.
Below there is the instruction how to get Jenkins in the Docker container with persistent configuration on GitHub.
Project structure
The simple way to get boilerplates of all files described in the post is to clone jenkins-in-docker repository. 

The repotisory contains Dockerfiles for master and agent Jenkins images:
$ git@github.com:antonfisher/jenkins-in-docker.git
$ cd jenkins-in-docker

Steps to get all working:
•	generate RSA keys for GitHub
•	build Jenkins master image
•	build Jenkins agent image
•	run Jenkins in a container
•	configure tasks.
Generate RSA keys
One key is for Jenkins configuration repository, another one for your application repository. Go to keys folder:
$ cd keys
Now let’s generate RSA keys for Jenkins master image. The first file is jenkins.config.id_rsa, this one will be used to access to Jenkins configuration on GitHub:
$ ssh-keygen -t rsa -b 4096 -C "my@gmail.com"
Generating public/private rsa key pair.
Enter file in which to save the key (/home/af/.ssh/id_rsa): jenkins.config.id_rsa

Then create git repository for Jenkins configuration on GitHub, in my case it is called my-jenkins-config. Add created key to GitHub (repository → Settings → Keys) with write access (!):
 
The second file is jenkins.application.id_rsa, this one will be used to pull your application code by Jenkins master container from GitHub:
$ ssh-keygen -t rsa -b 4096 -C "my@gmail.com"
Generating public/private rsa key pair.
Enter file in which to save the key (/home/af/.ssh/id_rsa): jenkins.application.id_rsa

Add this key to your application repository (repository → Settings → Keys).
We should have these files in the keys folder:
$ ls -l
total 16
-rw------- 1 af af 3243 Jan 15 18:39 jenkins.application.id_rsa
-rw-r--r-- 1 af af  743 Jan 15 18:39 jenkins.application.id_rsa.pub
-rw------- 1 af af 3243 Jan 15 18:33 jenkins.config.id_rsa
-rw-r--r-- 1 af af  743 Jan 15 18:33 jenkins.config.id_rsa.pub
Prepare Jenkins master Dockerfile
I used this configuration:
FROM jenkins:2.32.1
MAINTAINER Anton Fisher <a.fschr@gmail.com>

# root user for Jenkins, need to get access to /var/run/docker.sock (fix this in the future!)
USER root

# Environment
ENV HOME /root
ENV JENKINS_HOME /root/jenkins
ENV JENKINS_VERSION 2.32.1

# GitHub repository to store _Jenkins_ configuration
ENV GITHUB_USERNAME antonfisher
ENV GITHUB_CONFIG_REPOSITORY my-jenkins-config

# Make _Jenkins_ home directory
RUN mkdir -p $JENKINS_HOME

# Install _Jenkins_ plugins
RUN /usr/local/bin/install-plugins.sh \
    scm-sync-configuration:0.0.10 \
    workflow-aggregator:2.4 \
    docker-workflow:1.8

# Set timezone
RUN echo "America/Los_Angeles" > /etc/timezone &&\
    dpkg-reconfigure --frontend noninteractive tzdata &&\
    date

# Copy RSA keys for _Jenkins_ config repository (default keys).
# This public key should be added to:
# https://github.com/%YOUR_JENKINS_CONFIG_REPOSITORY%/settings/keys
COPY keys/jenkins.config.id_rsa     $HOME/.ssh/id_rsa
COPY keys/jenkins.config.id_rsa.pub $HOME/.ssh/id_rsa.pub
RUN chmod 600 $HOME/.ssh/id_rsa &&\
    chmod 600 $HOME/.ssh/id_rsa.pub
RUN echo "    IdentityFile $HOME/.ssh/id_rsa" >> /etc/ssh/ssh_config &&\
    echo "    StrictHostKeyChecking no      " >> /etc/ssh/ssh_config
RUN /bin/bash -c "eval '$(ssh-agent -s)'; ssh-add $HOME/.ssh/id_rsa;"

# Copy RSA keys for your application repository and add
# host 'github.com-application-jenkins' for application code pulls.
# This public key should be added to
# https://github.com/%YOUR_APPLICATION_REPOSITORY%/settings/keys
COPY keys/jenkins.application.id_rsa     $HOME/.ssh/jenkins.application.id_rsa
COPY keys/jenkins.application.id_rsa.pub $HOME/.ssh/jenkins.application.id_rsa.pub
RUN chmod 600 $HOME/.ssh/jenkins.application.id_rsa &&\
    chmod 600 $HOME/.ssh/jenkins.application.id_rsa.pub
RUN touch $HOME/.ssh/config &&\
    echo "Host github.com-application-jenkins                     " >> $HOME/.ssh/config &&\
    echo "    HostName       github.com                           " >> $HOME/.ssh/config &&\
    echo "    User           git                                  " >> $HOME/.ssh/config &&\
    echo "    IdentityFile   $HOME/.ssh/jenkins.application.id_rsa" >> $HOME/.ssh/config &&\
    echo "    IdentitiesOnly yes                                  " >> $HOME/.ssh/config

# Configure git
RUN git config --global user.email "jenkins@container" &&\
    git config --global user.name  "jenkins"

# Clone _Jenkins_ config
RUN cd /tmp &&\
    git clone git@github.com:$GITHUB_USERNAME/$GITHUB_CONFIG_REPOSITORY.git &&\
    cp -r $GITHUB_CONFIG_REPOSITORY/. $JENKINS_HOME &&\
    rm -r /tmp/$GITHUB_CONFIG_REPOSITORY

# _Jenkins_ workspace for sharing between containers
VOLUME $JENKINS_HOME/workspace

# Run init.sh script after container start
COPY src/init.sh /usr/local/bin/init.sh
ENTRYPOINT ["/bin/tini", "--", "/usr/local/bin/init.sh"]
Jenkins plugins are listed in the Dockerfile:
•	scm-sync-configuration:0.0.10 – to store Jenkins configuration in any SCM (link)
•	workflow-aggregator:2.4 – to enable Jenkins pipelines, task configuration (link)
•	docker-workflow:1.9 – to run Jenkins agents in Docker containers (link).
If you need some other plugins, here is the right place to add them.
We are ready to build Jenkins master container, there is a script to do this. The script just copies RSA keys to images/master/keys folder to use them during the build (this is Docker restrictions).
$ cd images/master
$ bin/image-build.sh

New Docker image will appear here:
$ docker images
REPOSITORY             TAG            IMAGE ID             CREATED               SIZE
jenkins-master         latest         a8f0c50f9ff6         2 minutes ago         719.3 MB

Run Jenkins
There is a script to run Jenkins:
$ cd images/master
$ bin/container-run.sh
During the first startup Jenkins requires initial setup, it provides temporary password in a console output:
...
Jenkins initial setup is required. An admin user has been created and a password generated.
Please use the following password to proceed to installation:

805cd12be95149e4b275fd71f6f0fcf1
...
Just copy the password and open http://localhost:8080 in the browser.

 

Go through installation but do NOT install any plugins in Jenkins installation wizard! Just skip this step. After that open Jenkins settings to configure synchronization with GitHub: http://localhost:8080/configure → "Configure System". There is my "SCM Sync configuration" section:

 

After some experiments, I added these files to "Manual synchronization includes" to also be synced with GitHub:
•	org.jenkinsci.plugins.workflow.flow.FlowExecutionList.xml
•	com.nirima.jenkins.plugins.docker.DockerPluginConfiguration.xml
•	nodes/*/config.xml
•	jobs/**/nextBuildNumber
•	secrets/jenkins.slaves.JnlpSlaveAgentProtocol.secret
•	secrets/master.key (be sure you do not use public repository like me :)
After pressing the "Save" button, new commits appears in the configuration repository on GitHub:

 

First task
Let’s create the first task to check synchronization and if it works after Jenkins restart.
 

Disaster happens
Now is time to simulate disaster, let’s imagine we lost the host with Docker containers. To simulate this just stop master container and remove its image:
$ docker stop jenkins-master
$ docker rm -f jenkins-master
$ docker rmi -f jenkins-master
To be sure do refresh http://localhost:8080/ – nothing is running.
Build image from scratch and run it again:
$ cd images/master
$ ./bin/image-build.sh
$ ./bin/container-run.sh
…open http://localhost:8080/, and here is our pre-configured Jenkins, no installation required:
 

The same workflow can be used for update Jenkins or its plugins.
Prepare Jenkins Agent Dockerfile
In my case it is plain agent that is based on Ubuntu 16.04, just do some image cleanup:
FROM ubuntu:16.04
MAINTAINER Anton Fisher <a.fschr@gmail.com>

USER root

# Set timezone
RUN echo "America/Los_Angeles" > /etc/timezone &&\
    dpkg-reconfigure --frontend noninteractive tzdata &&\
    date

# Set locale
RUN locale-gen en_US.UTF-8
ENV LANG en_US.UTF-8
ENV LANGUAGE en_US:en
ENV LC_ALL en_US.UTF-8

# Configure and update apt-get
ENV DEBIAN_FRONTEND "noninteractive"
RUN apt-get -q update &&\
    apt-get -q install -y -o Dpkg::Options::="--force-confnew" apt-utils &&\
    apt-get -q upgrade -y -o Dpkg::Options::="--force-confnew" --no-install-recommends

# Install dependencies
RUN apt-get -q install -y -o Dpkg::Options::="--force-confnew" \
    libltdl-dev \
    libltdl7 \
    sshpass \
    vim

# Clean-up apt-get
RUN apt-get -q autoremove &&\
    apt-get -q clean -y &&\
    rm -rf /var/lib/apt/lists/* &&\
    rm -f /var/cache/apt/*.bin

# Disable StrictHostKeyChecking for ssh
RUN echo "    StrictHostKeyChecking no" >> /etc/ssh/ssh_config

# staying online before force stop container
CMD ["tail", "-f", "/dev/null"]
Configure Jenkins tasks to use container-based agents
As I said above, I use pipelines to keep task configuration in the application code. You need to create a Jenkinsfile in your applications repository and then setup Jenkins to use it:

 

Let’s see an example:
node('master') {
    docker.withServer('unix:///var/run/docker.sock') {
        stage('Git clone') {
            git 'git@github.com-application-jenkins:antonfisher/node-mocha-extjs.git'
        }
        stage('Build') {
            docker
                .image('jenkins-agent-ubuntu')
                .inside('--volumes-from jenkins-master') {
                    sh "bash ./build.sh;"
                }
        }
        stage('Copy build results') {
            docker
                .image('jenkins-agent-ubuntu')
                .inside('--volumes-from jenkins-master') {
                    sh """
                        sshpass -plol scp \
                            "${WORKSPACE}/build/*.tar.gz" \
                            "backup@1.1.1.1:/buils";
                    """
                }
        }
        stage('UI unit tests') {
            docker
                .image('jenkins-agent-ubuntu')
                .inside('--volumes-from jenkins-master') {
                    sh """
                        cd ./tests;
                        bash ./run.sh;
                    """
                }
        }
    }
}

Steps:
•	node('master') {...} – use Jenkins master to run task
•	docker.withServer('unix:///var/run/docker.sock') {...} – run agent containers on master’s host
•	docker.image('jenkins-agent-ubuntu') – run "jenkins-agent-ubuntu" container
•	.inside('--volumes-from jenkins-master') {...} – use master’s volume (to share source code)
See docker in pipelines docs for more information.
Here is how pipelines may look in Jenkins:

 

Conclusion
Links
•	Repository with Jemkins master and agent Dockerfiles
•	Jenkins containers
•	Thanks for reading. I would be glad to get any feedback!
