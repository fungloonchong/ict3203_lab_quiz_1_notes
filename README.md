# Foreword

**THIS REVISION WAS RUSHED. IF YOU ARE UNSURE, USE THE MAIN BRANCH!**

This guide assumes:
1. You are comfortable with GNU/Linux commands.
2. BlueOcean is a glorified UI thus I did not use it.
3. You are running dind instead of passing the docker socket
4. You are using Linux (this guide was written with Ubuntu 22.04)
5. The user running all these commands belongs to the docker group (can run docker without sudo)
6. The result just has to be satisfactory. Any hacks used will be tolerated.

# Windows use

The environment variables like `$HOME` will be switched to `%HOMEDRIVE%%HOMEPATH%` and `\` characters that was used for new line will be replaced with `^`.

For more detailed use case, please refer `https://www.jenkins.io/doc/tutorials/build-a-node-js-and-react-app-with-npm/`

# Extracting data volumes from CDN

**WARNING**

If you are unpacking data volumes from CDN, you **MUST** do these steps before running their respective containers.

If you want to setup your own Jenkins (incl. OWASP DepCheck & Maven & SonarQube plugin) & SonarQube, you can safely skip to other sections such as `Lab X05`.

```bash
docker run \
  --rm \
  --name ubuntu \
  --volume jenkins-data:/var/jenkins_home \
  --volume sonarqube-data:/opt/sonarqube/data \
  -it \
  ubuntu:latest \
  bash
```

After running the command above, **DO NOT CLOSE THIS TERMINAL!!**

Switch to another terminal and continue with the commands below.

```bash
docker exec -u 0 -it ubuntu sh -c "apt update && apt install wget tar -y"
docker exec -u 0 -it ubuntu sh -c "wget -c https://cflcdn.sgp1.digitaloceanspaces.com/sit/ict3203/lab_quiz_1/jenkins_home.tar.gz -O - | tar -pzxvf -"
docker exec -u 0 -it ubuntu sh -c "wget -c https://cflcdn.sgp1.digitaloceanspaces.com/sit/ict3203/lab_quiz_1/sonarqube_data.tar.gz -O - | tar -pzxvf -"
docker container kill ubuntu
```

## Starting the containers
```bash
docker network create jenkins
```

### DIND

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
  --publish 3000:3000 \
  --publish 5000:5000 \
  docker:dind \
  --storage-driver overlay2
```

### BlueOcean
```bash
cd ~/
mkdir blueocean && cd blueocean
```

Use an editor of your choice and paste the Dockerfile in and save it as `Dockerfile`

```Dockerfile
FROM jenkins/jenkins:2.361.3-jdk11
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
RUN jenkins-plugin-cli --plugins "blueocean:1.25.8 docker-workflow:521.v1a_a_dd2073b_2e"
```

```bash
cat Dockerfile
# if all is good, build the image
docker build -t myjenkins-blueocean:2.361.3-1 .
```

```bash
docker run \
  --name jenkins-blueocean \
  --detach \
  --network jenkins \
  --env DOCKER_HOST=tcp://docker:2376 \
  --env DOCKER_CERT_PATH=/certs/client \
  --env DOCKER_TLS_VERIFY=1 \
  --publish 8080:8080 \
  --publish 50000:50000 \
  --volume jenkins-data:/var/jenkins_home \
  --volume jenkins-docker-certs:/certs/client:ro \
  --volume "$HOME":/home \
  --restart=on-failure \
  --env JAVA_OPTS="-Dhudson.plugins.git.GitSCM.ALLOW_LOCAL_CHECKOUT=true" \
  myjenkins-blueocean:2.361.3-1
```

### SonarQube
```
docker run \
  -d \
  --name sonarqube \
  --network jenkins \
  -e SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true \
  -p 9000:9000 \
  --volume sonarqube-data:/opt/sonarqube/data \
  sonarqube:latest
```

## Credentials/token
```bash
# jenkins credentials
# line 1 is the username
# line 2 is the password
# the username and password is the same
docker exec -it jenkins-blueocean sh -c 'cat /var/jenkins_home/CREDENTIALS'

# sonarqube credentials
# line 1 is the username
# line 2 is the password
docker exec -it sonarqube sh -c 'cat /opt/sonarqube/data/CREDENTIALS'

# sonarqube secret token
docker exec -it sonarqube sh -c 'cat /opt/sonarqube/data/TOKEN'
```

## Dev notes

You can safely ignore this section. Its just my notes for developing this solution.

```bash
# Note to self (dev)
# create tarballs
tar -pczvf /jenkins_home.tar.gz /var/jenkins_home/
tar -pczvf /sonarqube_data.tar.gz /opt/sonarqube/data/

# ls tarballs
tar tvf /jenkins_home.tar.gz
```

# Lab X05

Adapted from: `https://www.jenkins.io/doc/tutorials/build-a-node-js-and-react-app-with-npm/`

## Create network bridge

```bash
docker network create jenkins
```

## Start DIND

Start this on host

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
  --publish 3000:3000 \
  --publish 5000:5000 \
  docker:dind \
  --storage-driver overlay2
```

## Creating Dockerfile for BlueOcean and building it

```bash
cd ~/
mkdir blueocean && cd blueocean
```

Use an editor of your choice and paste the Dockerfile in and save it as `Dockerfile`

```Dockerfile
FROM jenkins/jenkins:2.361.3-jdk11
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
RUN jenkins-plugin-cli --plugins "blueocean:1.25.8 docker-workflow:521.v1a_a_dd2073b_2e"
```

```bash
cat Dockerfile
# if all is good, build the image
docker build -t myjenkins-blueocean:2.361.3-1 .
```

## Running BlueOcean
```bash
docker run \
  --name jenkins-blueocean \
  --detach \
  --network jenkins \
  --env DOCKER_HOST=tcp://docker:2376 \
  --env DOCKER_CERT_PATH=/certs/client \
  --env DOCKER_TLS_VERIFY=1 \
  --publish 8080:8080 \
  --publish 50000:50000 \
  --volume jenkins-data:/var/jenkins_home \
  --volume jenkins-docker-certs:/certs/client:ro \
  --volume "$HOME":/home \
  --restart=on-failure \
  --env JAVA_OPTS="-Dhudson.plugins.git.GitSCM.ALLOW_LOCAL_CHECKOUT=true" \
  myjenkins-blueocean:2.361.3-1
```

## Executing bash in docker containers
```bash
# docker exec -it <CONTAINER_NAME OR ID> <COMMAND>
# use bash first for a better shell. if it doesn't work, try sh
docker exec -it jenkins-blueocean bash
```

## Copying files between containers and/or host
```bash
# docker cp [<SRC_CONTAINER>:<PATH>] [<DST_CONTAINER>:<PATH>]
#
# container to container
docker cp jenkins-blueocean:/tmp/example.txt jenkins-docker:/tmp
#
# host to container
docker cp ./example.txt jenkins-blueocean:/tmp
#
# container to host
docker cp jenkins-blueocean:/tmp/example.txt ./example2.txt 
```

## Getting the logs for Jenkins
```bash
docker logs jenkins-blueocean

# you can see the admin password for the initial setup
```

## Initial setup

Access `http://localhost:8080` and use the password found above to setup Jenkins.

Follow the steps as shown in the setup to continue.

## Optional: stop/remove container

```bash
# get list of containers
docker ps

# stop containers
#docker container kill <CONTAINER ID OR NAME>
docker container kill jenkins-blueocean

# remove containers (requires the container to be killed)
# only required if --rm switch is not used to run the container
docker container rm jenkins-blueocean
```

## Fork repo
```bash
# on your host's home directory as you have binded /home
cd ~/
git clone https://github.com/jenkins-docs/simple-node-js-react-npm-app
```

## Creating a Pipeline project
Click `New Item`, fill in a name, select `Pipeline` and `OK`.

Scroll down to `Pipeline` section and change the `Definition` to `Pipeline script from SCM`.

Select `Git` as the `SCM` and input `/home/simple-node-js-react-npm-app` as the `Repository URL`. Leave the rest as default and click `Save`.

**NOTE** Your path for the repository URL may be different but you can assume the path given as above.

**NOTE2** You may need to have the environment variable `JAVA_OPTS` set to `-Dhudson.plugins.git.GitSCM.ALLOW_LOCAL_CHECKOUT=true`. Look at the section `Creating Dockerfile for BlueOcean and building it`.

## Creating Jenkinsfile

Now that you have the repo, we can create our Jenkinsfile.

Add the content below into a new file with the name `Jenkinsfile` at the root directory of the repository.

```Groovy
pipeline {
  agent {
    docker {
      image 'node:lts-bullseye-slim' 
      args '-p 3000:3000' 
    }
  }
  stages {
    stage('Build') { 
      steps {
        sh 'npm install' 
      }
    }
  }
}
```

After doing so, remember to commit it by running:
```bash
git add .
git commit -m "Added Jenkinsfile"
```

Now, we can return to our Jenkins webpage and click on `Build Now` on our Jenkins job.

**Note** if you fail this step, its best to check the `Console Output` of the build instead of checking the console output of the failed step.

**Note2** it might be possible that you have not mounted the home directory. You can check that by running `docker exec -it <CONTAINER NAME OR ID> <COMMAND>`. For more details, refer to `Executing bash in docker containers`

If there are no issues, it should be a successful build.

## Adding test stage
Going back to the file `Jenkinsfile` and replacing it with the content below.

```Groovy
pipeline {
  agent {
    docker {
      image 'node:lts-bullseye-slim'
      args '-p 3000:3000'
    }
  }
  stages {
    stage('Build') {
      steps {
        sh 'npm install'
      }
    }
    stage('Test') { 
      steps {
        sh './jenkins/scripts/test.sh' 
      }
    }
  }
}
```

After doing so, remember to commit it by running:
```bash
git add .
git commit -m "Added test stage"
```

The build should pass without issue. If not, please refer to the troubleshooting steps as describe in the stage above this.

## Adding the final deliver stage
Going back to the file `Jenkinsfile` and replacing it with the content below.

```Groovy
pipeline {
  agent {
    docker {
      image 'node:lts-bullseye-slim'
      args '-p 3000:3000'
    }
  }
  stages {
    stage('Build') {
      steps {
        sh 'npm install'
      }
    }
    stage('Test') { 
      steps {
        sh './jenkins/scripts/test.sh' 
      }
    }
    stage('Deliver') {
      steps {
        sh './jenkins/scripts/deliver.sh'
        input message: 'Finished using the web site? (Click "Proceed" to continue)'
        sh './jenkins/scripts/kill.sh'
      }
    }
  }
}
```

After doing so, remember to commit it by running:
```bash
git add .
git commit -m "Added deliver stage"
```

The build is unlikely to pass without issue. Refer to the section `Webpack failed to build with ERR_OSSL_EVP_UNSUPPORTED` to add the necessary environment variable.

**Note** In the event that Jenkins mentions about invalid permission, you might need to add the step `sh 'chmod +x ./my-script'` before running any `.sh` script.

**Note 2** Refer to the troubleshooting step above for any issues

Access the web page `http://localhost:3000` and you should be seeing a React logo. If the page does not load, you may want to check whether you have published port 3000 from dind container.

After you are done accessing, you can click on `Proceed` on the build page to terminate the react container.

## Editing the React page (hot-reloading)
Maybe you know of other tools to do the job but you can follow the steps as described in `Copying files between containers and/or host` to obtain similar results.

Or you can edit the file from shell.

```bash
docker exec -it jenkins-docker sh
# my project name is test1
vi /var/jenkins_home/workspace/test1/src
```

Save it and the changes should reflect without need a rebuild.

# Final Jenkinsfile

```Groovy
pipeline {
  agent {
    docker {
      image 'node:lts-buster-slim'
      args '-p 3000:3000'
    }
  }
  environment {
    NODE_OPTIONS = '--openssl-legacy-provider'
  }
  stages {
    stage('Build') {
      steps {
        sh 'npm install'
      }
    }
    stage('Test') {
      steps {
        sh './jenkins/scripts/test.sh'
      }
    }
    stage('Deliver') { 
      steps {
        sh './jenkins/scripts/deliver.sh' 
        input message: 'Finished using the web site? (Click "Proceed" to continue)' 
        sh './jenkins/scripts/kill.sh' 
      }
    }
  }
}
```

# Lab X05: Errors & mitigation

## Webpack failed to build with ERR_OSSL_EVP_UNSUPPORTED

Adapted from: `https://stackoverflow.com/questions/72866798/node-openssl-legacy-provider-is-not-allowed-in-node-options`

```bash
export NODE_OPTIONS=--openssl-legacy-provider
```

# Lab X06

Adapted from: `https://medium.com/nagarro/sec-in-your-devops-adding-the-owasp-dependency-check-to-your-jenkins-pipeline-2f005827583d`

This assumes your Jenkins is already up. If not, refer to `Lab X05` for the steps.

## Install OWASP Dependency-Check on Jenkins

From `Dashboard`, click `Manage Jenkins` on the menu located on the left of the page.

Under `System Configuration` section, click `Manage Plugins`.

Click on `Available` tab and search for `OWASP Dependency-Check`.

Check the box and click `Download now and install after restart`.

After the download has been completed, click on `Restart Jenkins when installation is complete and no jobs are running`.

In the event that Jenkins does not restart, you might need to start it again on your own.

## Configuring OWASP Dependency-Check

From `Dashboard`, click `Manage Jenkins` on the menu located on the left of the page.

Under `System Configuration` section, click `Global Tool Configuration`.

Scroll down to find the section `Dependency-Check`. As this is a fresh install, there should not any installations.

Click `Add Dependency-Check` and fill the `Name` with `Default`. If you set any other name, you will have to change the key `odcInstallation` on the Jenkinsfile below.

Thereafter, click `Save`.

## Cloning the repository

```bash
cd ~/
git clone https://github.com/0xprime/JenkinsDependencyCheckTest
```

## Creating the Jenkins job
From `Dashboard`, click `New Item`.

Fill in a name, select `Pipeline` and click `OK`.

Go to the `Pipeline` section and change the `Definition` to `Pipeline script from SCM`.

Change `SCM` to `Git` and input the path of the repository in `Repository URL`. For me, its `/home/JenkinsDependencyCheckTest`

## Creating the Jenkinsfile

Now, we need to create the `Jenkinsfile`. Add the following content to the file `Jenkinsfile`.

```Groovy
pipeline {
  agent any
  stages {
    stage('OWASP Check') {
      steps {
        dependencyCheck additionalArguments: '--format HTML --format XML', odcInstallation: 'Default'
      }
    }
  }
  post {
    always {
      dependencyCheckPublisher pattern: 'dependency-check-report.xml'
    }
  }
}
```

After doing so, remember to commit it by running:
```bash
git add .
git commit -m "Added initial Jenkinsfile"
```

## Running the Pipeline
Go back to your job's page on Jenkins.

Click on `Build Now`. This will take awhile.

The build should be successful. Please troubleshoot with the steps described in `Lab X05` if required.

Click on the latest build under `Build History` and click `Dependency-Check`.

We can see there are 4 vulnerabilities present. You can click on any of the vulnerabilities to expand it.

## Detailed report

For a more detailed view, we can click on `Workspaces` and selecting the first path on the list.

As there are issues opening the html report without downloading, we can right click `dependency-check-report.html` and download it. Now, we can open it and the report will be displayed correctly.


## Suppressing a vulnerability
If we would want to whitelist/suppress a CVE, we can find the CVE and click `suppress` to the right of the CVE number.

A pop-up would appear. Copy the highlighted text and save it to the file `suppression.xml`. An example of the highlighted text is shown below.

```xml
<suppress>
   <notes><![CDATA[
   file name: jquery-2.1.4.min.js
   ]]></notes>
   <packageUrl regex="true">^pkg:javascript/jquery@.*$</packageUrl>
   <cve>CVE-2019-11358</cve>
</suppress>
```

But this is not the correct format. So let's add the missing tags as shown below.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<suppressions xmlns="https://jeremylong.github.io/DependencyCheck/dependency-suppression.1.3.xsd">
  <suppress>
    <notes><![CDATA[
    file name: jquery-2.1.4.min.js
    ]]></notes>
    <packageUrl regex="true">^pkg:javascript/jquery@.*$</packageUrl>
    <cve>CVE-2019-11358</cve>
  </suppress>
</suppressions>
```

Now that we have a suppression list, we need to update our Jenkinsfile.

```Groovy
pipeline {
  agent any
  stages {
    stage('OWASP Check') {
      steps {
        dependencyCheck additionalArguments: '--format HTML --format XML --suppression suppression.xml', odcInstallation: 'Default'
      }
    }
  }
  post {
    always {
      dependencyCheckPublisher pattern: 'dependency-check-report.xml'
    }
  }
}
```

After doing so, remember to commit it by running:
```bash
git add .
git commit -m "Added suppression list"
```

## Running the Pipeline (with suppression list)
Go back to your job's page on Jenkins.

Click on `Build Now`. This should be much faster than before.

After the build, you should see less than 4 vulnerabilities.

# Lab X07A

Adapted from: `https://www.youtube.com/watch?v=68cDNUz7uro` but I don't see why you would want to use YouTube during the lab test.

Assumes that you have the files in the host already.

Assumes your host is Ubuntu Linux.

## Optional: change all line endings to be unix

```bash
# to prevent \r\n messing up the deployment.
# we will standardize and use \n
sudo apt install -y dos2unix
```

## Unpacking the zip file & git init

```bash
unzip jenkins-phpunit-test.zip -d ./
cd jenkins-phpunit-test
find ./ -type f -exec dos2unix {} +; # run this to convert dos line ending to unix. ignore the errors
git init
git add . 
git commit -m "Added initial files"
```

## Create the new job on Jenkins

Use either `Lab X05` or `Lab X06` to create the job.

Click `Build Now` after creating the job.

The build should be successful.

Click on the build output to verify that there is a line `OK (1 test, 1 assertion)`

## Enhancing the Jenkins file

As instructed, we need to add more stuff such as updating the test stage and have a post build steps.


### Before
```Groovy
pipeline {
  agent {
    docker {
      image 'composer:latest'
    }
  }
  stages {
    stage('Build') {
      steps {
        sh 'composer install'
      }
    }
    stage('Test') {
      steps {
        sh './vendor/bin/phpunit tests'
      }
    }
  }
}
```

### After
```Groovy
pipeline {
  agent {
    docker {
      image 'composer:latest'
    }
  }
  stages {
    stage('Build') {
      steps {
        sh 'composer install'
      }
    }
    stage('Test') {
      steps {
        sh './vendor/bin/phpunit --log-junit logs/unitreport.xml -c tests/phpunit.xml tests'
      }
    }
  }
  post {
    always {
      junit testResults: 'logs/unitreport.xml'
    }
  }
}
```

After doing so, remember to commit it by running:
```bash
git add .
git commit -m "Enhanced Jenkinsfile"
```

Click `Build Now` after making the commit.

The build should be successful.

Now, you can see a new option `Test Result` on the menu bar on the left.

Click into it and we can see there are a total of 1 tests and 1 passed.

## Introducing bug to codebase

Now we need to intentionally mess with the function` turnWheel()` in `GumballMachine.php`

### Before
```php
public function turnWheel() {
  $this->setGumballs($this->getGumballs()-1);
}
```

### After
```php
public function turnWheel() {
  $this->setGumballs($this->getGumballs()-2);
}
```

After doing so, remember to commit it by running:
```bash
git add .
git commit -m "Enhanced Jenkinsfile"
```

Click `Build Now` after making the commit.

The build should be unsuccessful.

Let's navigate to `Test Result` on the menu bar on the left.

Click into it and we can see there are a total of 1 tests, 0 passed and 1 failed.

This is the expected result.

# Lab X07B

Adapted from: `https://www.wdb24.com/session-in-php-example-for-login-logout/`

Assumes that you have the files in the host already.

Assumes your host is Ubuntu Linux.

## Optional: change all line endings to be unix

```bash
# to prevent \r\n messing up the deployment.
# we will standardize and use \n
sudo apt install -y dos2unix
```

## Unpacking the zip file & git init

```bash
unzip jenkins-php-selenium-test.zip -d ./
cd jenkins-php-selenium-test
find ./ -type f -exec dos2unix {} +; # run this to convert dos line ending to unix. ignore the errors
git init
#git update-index --chmod=+x jenkins/scripts/deploy.sh
#git update-index --chmod=+x jenkins/scripts/kill.sh
# the two commands did not worked for me. the bottom command works.
chmod +x jenkins/scripts/*.sh
```

### Problems in deploy.sh

Fixing the problems in the `jenkins/script/deploy.sh`. Change it to the below.
Problems:
1. It assumes you are using Windows
2. It wasn't scripted with dind in mind.

Fix:
1. Assumes you are using the steps in `Lab X05` where you mount `jenkins-data` volume to your dind
2. Assumes your Jenkins job name is `test4`
3. Create a new docker network `test4net`

### New deploy.sh
```bash
#!/usr/bin/env sh

set -x
docker run -d -p 5000:80 --name my-apache-php-app -v /var/jenkins_home/workspace/test4/src:/var/www/html --net test4net php:7.2-apache
sleep 1
set +x

echo 'Now...'
echo 'Visit http://localhost:5000 to see your PHP application in action.'
```

### New Jenkinsfile

```Groovy
pipeline {
  agent none
  stages {
    stage('Integration UI Test') {
      parallel {
        stage('Deploy') {
          agent any
          steps {
            sh './jenkins/scripts/deploy.sh'
            input message: 'Finished using the web site? (Click "Proceed" to continue)'
            sh './jenkins/scripts/kill.sh'
          }
        }
        stage('Headless Browser Test') {
          agent {
            docker {
              image 'maven:3-alpine' 
              args '-v /root/.m2:/root/.m2 --net="test4net"' 
            }
          }
          steps {
            sh 'mvn -B -DskipTests clean package'
            sh 'mvn test'
          }
          post {
            always {
              junit 'target/surefire-reports/*.xml'
            }
          }
        }
      }
    }
  }
}
```

### Updated AppTest.java

Located at `src/test/java/mycompany/app/AppTest.java` of `jenkins-php-selenium-test`.

**DO NOT COPY EVERYTHING. NOTE THE LINE TO CHANGE**

```java
public class AppTest
{
  WebDriver driver;
  WebDriverWait wait;
  String url = "http://my-apache-php-app"; // <- Change this only
  String validEmail = "user@example.com";
  String validPassword = "password1234";
  String invalidEmail = "none@example.com";
  String invalidPassword = "password";
}
```

### Create new docker network
```bash
docker exec -it jenkins-docker sh -c 'docker network create test4net'
```

### Commit to git

```bash
git add . 
git commit -m "Added initial files"
```

Commit to git.

## Create the new job on Jenkins

Use either `Lab X05` or `Lab X06` to create the job.

Click `Build Now` after creating the job. This will take awhile. You can monitor the progress by going to the build output.

The test may be done if the progress look like it is stuck. Near the bottom, there should be a line that says `Tests run: 2, Failures: 0, Errors: 0, Skipped: 0`. This would mean it is successful.

Now we can terminate the build, there should be a button near the middle of the build output. Click `Proceed` to terminate the test.

## Clean up
You should remove the network that is not used in other labs. Removal is optional as it does not cause any major issue.

```bash
docker exec -it jenkins-docker sh -c 'docker network rm test4net'
```

# Lab X08

Adapted from:
- `https://github.com/jenkinsci/warnings-ng-plugin/blob/master/doc/Documentation.md`
- `https://github.com/ScaleSec/vulnado`

## Installing the plugin
From `Dashboard`, click `Manage Jenkins` on the menu located on the left of the page.

Under `System Configuration` section, click `Manage Plugins`.

Click on `Available` tab and search for `Warnings Next Generation` plugin.

Check the box and click `Download now and install after restart`.

After the download has been completed, click on `Restart Jenkins when installation is complete and no jobs are running`.

In the event that Jenkins does not restart, you might need to start it again on your own.

## Downloading Maven onto Jenkins

```bash
wget -O apache-maven-3.6.3-bin.tar.gz http://mirrors.estointernet.in/apache/maven/maven-3/3.6.3/binaries/apache-maven-3.6.3-bin.tar.gz
tar -zxvf apache-maven-3.6.3-bin.tar.gz
docker cp ./apache-maven-3.6.3 jenkins-blueocean:/var/jenkins_home/
```

You can check it by running the following
```bash
docker exec -it jenkins-blueocean sh -c 'ls -la /var/jenkins_home/apache-maven-3.6.3'
```

## Installing Maven in Jenkins

From `Dashboard`, click `Manage Jenkins` on the menu located on the left of the page.

Under `System Configuration` section, click `Manage Plugins`.

Click on `Available` tab and search for `Maven`.

Select `Maven Integration` & `Maven Invoker`

Check the box and click `Download now and install after restart`.

After the download has been completed, click on `Restart Jenkins when installation is complete and no jobs are running`.

In the event that Jenkins does not restart, you might need to start it again on your own.

## Configuring Maven in Jenkins

From `Dashboard`, click `Manage Jenkins` on the menu located on the left of the page.

Under `System Configuration` section, click `Global Tool Configuration`.

Scroll down to find the section `Maven` (not Maven Configuration) and click `Add Maven`.

Enter `Maven` for the `Name`, uncheck `Install automatically`, enter the path for `MAVEN_HOME` as `/var/jenkins_home/apache-maven-3.6.3` and hit `Save`.

## Creating the Jenkinsfile

```Groovy
pipeline {
  agent any
  stages {
    stage ('Checkout') {
      steps {
        git branch:'master', url: 'https://github.com/ScaleSec/vulnado.git'
      }
    }
    stage ('Build') {
      steps {
        sh '/var/jenkins_home/apache-maven-3.6.3/bin/mvn --batch-mode -V -U -e clean verify -Dsurefire.useFile=false -Dmaven.test.failure.ignore'
      }
    }
    stage ('Analysis') {
      steps {
        sh '/var/jenkins_home/apache-maven-3.6.3/bin/mvn --batch-mode -V -U -e checkstyle:checkstyle pmd:pmd pmd:cpd findbugs:findbugs'
      }
    }
  }
  post {
    always {
      junit testResults: '**/target/surefire-reports/TEST-*.xml'
      recordIssues enabledForFailure: true, tools: [mavenConsole(), java(), javaDoc()]
      recordIssues enabledForFailure: true, tool: checkStyle()
      recordIssues enabledForFailure: true, tool: spotBugs(pattern:'**/target/findbugsXml.xml')
      recordIssues enabledForFailure: true, tool: cpd(pattern: '**/target/cpd.xml')
      recordIssues enabledForFailure: true, tool: pmdParser(pattern: '**/target/pmd.xml')
    }
  }
}
```

## Create the new job on Jenkins

Use either `Lab X05` or `Lab X06` to create the job but we will be using `Pipeline script` instead of `Pipeline script from SCM`. Paste the above Jenkins file into it

Click `Build Now` after creating the job. This will take a long while. You can monitor the progress by going to the build output.

After the build is completed, click on `Status`.

We can see 8 Maven warnings, no JAva compiler warnings, no JavaDoc warnings, 182 CheckStyle warnings, 19 SpotBugs warning, no CPD warnings and 12 PMD warnings.

# Lab X09

Adapted from: `https://docs.sonarqube.org/latest/setup/get-started-2-minutes/`

Requires:
- SAST (X08)

## Installing SonarQube Scanner in Jenkins

From `Dashboard`, click `Manage Jenkins` on the menu located on the left of the page.

Under `System Configuration` section, click `Manage Plugins`.

Click on `Available` tab and search for `SonarQube Scanner`.

Check the box and click `Download now and install after restart`.

After the download has been completed, click on `Restart Jenkins when installation is complete and no jobs are running`.

In the event that Jenkins does not restart, you might need to start it again on your own.

## Downloading & running the SonarQube image from Docker Hub
Run this command on your host.

```bash
# i think this is redundant
docker pull sonarqube:latest

docker run \
  -d \
  --name sonarqube \
  -e SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true \
  -p 9000:9000 \
  --volume sonarqube-data:/opt/sonarqube/data \
  sonarqube:latest

docker network connect jenkins sonarqube

# optional
docker network inspect jenkins
```

Access the webpage at `http://localhost:9000`. This might take awhile.

## Login

Default credentials:
Username: `admin`
Password: `admin`

Thereafter, you will be prompted to change the password.

## Create a new SonarQube project

After logging in and changing the password, you will be presented the landing page.
Click on `Manually` to create a SonarQube project.

Fill in the `Project display name` as `OWASP` and `Project key` as `OWASP`. Click `Set Up`.

Click `Locally`.

Generate a new token for this project by clicking "Generate". You can change the `Token Name` if you wish to do so.

Copy the token or paste it on a notepad before clicking `Continue`.

For the main language, click `Other (JS, TS, Go, Python, PHP, ...)` and the OS to be `Linux`.

## Continuing from Jenkins

### Adding token

Now head back to Jenkins.

From `Dashboard`, click `Manage Jenkins` on the menu located on the left of the page.

Under `Security` section, click on `Manager Credentials`.

Hover over `(global)`, and click `Add credentials`.

Select `Secret text` for `Kind`. Paste your token under `Secret`. It is fine to leave the rest as the defaults.

### Configuring SonarQube URL

Under `System Configuration` section, click `Configure System`.

Scroll to find the section `SonarQube servers`. Click `Add SonarQube`, put `SonarQube` for the Name, and the server URL `http://sonarqube:9000`.

For the `Server authentication token`, click on the dropdown menu and select the first option. It should be `Secret text`.

Click `Save`.

## Configuring SonarQube Scanner

From `Dashboard`, click `Manage Jenkins` on the menu located on the left of the page.

Under `System Configuration` section, click on `Global Tool Configuration`.

Scroll down to find the section `SonarQube Scanner` and click `Add SonarQube Scanner`.

Input `SonarQube` under `Name` and change the version to `SonarQube Scanner 4.6.2.2472`, or at least that is the version that is suggested by prof.

Click `Save`.

## Creating the Jenkinsfile

```Groovy
pipeline {
  agent any
  stages {
    stage ('Checkout') {
      steps {
        git branch:'master', url: 'https://github.com/OWASP/Vulnerable-Web-Application.git'
      }
    }
    stage('Code Quality Check via SonarQube') {
      steps {
        script {
          def scannerHome = tool 'SonarQube'; // <- Name of your SonarQube in Jenkins' System Configuration
            withSonarQubeEnv('SonarQube') { // <- Same as above?
              sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=OWASP -Dsonar.sources=." // <- Project key should be the name of your project on SonarQube's webpage
            }
        }
      }
    }
  }
  post {
    always {
      recordIssues enabledForFailure: true, tool: sonarQube()
    }
  }
}
```

## Create the new job on Jenkins

Use either `Lab X05` or `Lab X06` to create the job but we will be using `Pipeline script` instead of `Pipeline script from SCM`. Paste the above Jenkins file into it

Click `Build Now` after creating the job. This will take awhile. You can monitor the progress by going to the build output.

After the build is completed, head back to the SonarQube webpage and you should be able to see the project `OWASP` (or whatever name you have changed it to). Click on it and you can see the detailed breakdown of the issues.

## Optional: Running SonarQube Scanner as a standalone

```bash
# SONAR_LOGIN is the token
# projectKey is the name of your SonarQube project
docker run --rm -e SONAR_HOST_URL=http://192.168.16.128:9000 -e SONAR_LOGIN=2742fdcd3a1fe63a0912d32ebd77a1c74a4e212d -it -v "$(pwd):/usr/src" sonarsource/sonar-scanner-cli -Dsonar.projectKey=OWASP
```

## Optional: Increasing SonarQube's memory limit
```bash
docker run --memory=3g <ARGS/SWITCHES>

# example of this section with the docker run command
docker run --rm -e SONAR_HOST_URL=http://192.168.16.128:9000 -e SONAR_LOGIN=2742fdcd3a1fe63a0912d32ebd77a1c74a4e212d -it -v "$(pwd):/usr/src" --memory=3g sonarsource/sonar-scanner-cli -Dsonar.projectKey=OWASP
```

# Lab X10

## Compose file

Adapted from: `https://docs.docker.com/compose/compose-file/`

To start the `docker-compose.yml` in the cwd.

```bash
docker-compose up
```

To stop the `docker-compose.yml` in the cwd.

```bash
docker-compose down
```

Sections that can be configured:
- Version
- Services
- Network
- Config
- Secrets

## Adding the nginx service

### Compose file

```yml
services:
  nginxwebsvr:
    image: nginx:alpine
    container_name: nginxwebsvr
    ports:
      - "8000:80"
```

### Starting the service(s)

```bash
docker-compose up
```

### Check if the services are up

```bash
docker ps --all
```

### Accessing the service

Visit the URL `http://localhost:8080/`.


## Adding the MySQL service

### Compose file

Replace the compose file with the following

```yml
services:
  nginxwebsvr:
    image: nginx:alpine
    container_name: nginxwebsvr
    ports:
      - "8000:80"
  mysqldb:
    image: mysql:5.7
    container_name: mysqlsvr
    restart: always
    volumes:
      - ./mysql_data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: pass
      MYSQL_DATABASE: testdb
      MYSQL_USER: user
      MYSQL_PASSWORD: pass
```

### Starting the service(s)

```bash
docker-compose up
```

### Check if the services are up

```bash
docker ps --all
```

### Accessing the service

```bash
docker exec -it mysqlsvr sh -c 'mysql -u root -p'
# login with the password. it is `pass` if you did not change it
# once you are in the mysql shell, you can type `\q` to quit
```

## Git Server (w/o authentication)

```bash
mkdir ./repos/
```

Paste the content below into `git1.Dockerfile`

```Dockerfile
FROM node:alpine

RUN apk add --no-cache tini git \
  && yarn global add git-http-server \
  && adduser -D -g git git

USER git
WORKDIR /home/git

RUN git init --bare repository.git

ENTRYPOINT ["tini", "--", "git-http-server", "-p", "3000", "/home/git"]
```


Replace the content below in `docker-compose.yml`
```yml
services:
  nginxwebsvr:
    image: nginx:alpine
    container_name: nginxwebsvr
    ports:
      - "8000:80"
  mysqldb:
    image: mysql:5.7
    container_name: mysqlsvr
    restart: always
    volumes:
      - ./mysql_data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: pass
      MYSQL_DATABASE: testdb
      MYSQL_USER: user
      MYSQL_PASSWORD: pass
  git-server:
    container_name: git1svr
    build:
      dockerfile: git1.Dockerfile
      context: .
    restart: always
    ports:
      - "3000:3000"
    volumes:
      - ./repos:/var/www/git
```

### Starting the service(s)

```bash
docker-compose build
docker-compose up
# or. the command below will start it detached.
docker-compose up -d
```

### Check if the services are up

```bash
docker ps --all
```

### Accessing the service

```bash
git clone http://localhost:3000/repository.git
```

You will probably see:

`Cloning into 'repository'...`

`warning: You appear to have cloned an empty repository.`

## Git Server (w/ SSH)

I am too lazy.

Maybe this is useful enough?
- `https://github.com/6arms1leg/git-ssh-docker`
- `https://github.com/jkarlosb/git-server-docker/blob/master/Dockerfile`

Therefore, the sections below may be incomplete.

### Generate SSH keys

**PLEASE KNOW WHAT YOU ARE DOING. THIS ACTION MAY OVERRIDE YOUR SSH KEY ON YOUR HOST**

If it matters that is.

```bash
ssh-keygen -f ./example -N ''
```

### Running the container
```bash
docker run \
  --name git2ssh \
  -d \
  -p 2222:22 \
  -v ~/git-server/keys:/git-server/keys \
  -v ~/git-server/repos:/git-server/repos \
  jkarlos/git-server-docker
```

### Copying your SSH key over
```bash
# you actually need sudo for this because the container is probably privileged?
sudo cp ./example.pub ~/git-server/keys/ 
```

### Restarting the container
```bash
docker restart git2ssh
```

This will take awhile.

### Accessing the repo
```bash
ssh git@localhost -p 2222 -i ./example
# you will have to accept the signature by typing `yes` followed by enter.
```
