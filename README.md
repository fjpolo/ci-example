# Embedded ci-example

This README will walk you an example CI pipeline. In fact, TWO pipelines in one: (1) for a Rich IoT Company developing a python application, and (2) for a Low-Power IoT Company developing a bare-metal application in C. Going through this walkthrough will shed light on how to create a basic CI pipeline for either type of embedded software development, quickly.

After going through this README and examining this repo's code you will have a working CI pipeline example for bare-metal and embedded linux applications, including:
- Setting up a CI server (with Jenkins)
- Connecting the CI server to a source control repo (with GitHub)
- Creating isolated slave environments (with Docker containers)
- Creating a build stage in a pipeline and compiling bare-metal software (with Arm Compiler 6)
- Passing artifacts bewteen CI stages (like a compiled application from the build stage to the test stage)
- Creating a test stage in a pipeline and running tests for both bare-metal software and embedded linux software (with Arm Fast Models and Docker containers)
- Gathering test data into machine readable results (via Jenkins and bash scripting)

It is recommended to be at least familer with GitHub, Docker, and CI before starting this walkthrough. Other concepts will be explained as they arise, like navigating Jenkins and setting up Arm tools.

Let's jump in.

## Machine Setup
I created this example on a fresh Linux machine (using AWS EC2, but you can use another cloud provider, Virtual Machine, or a personal machine just as well). My OS is Ubuntu 18.04, the below examples should work for most Ubuntu versions (but I wouldn't go lower than 16.04). Do a apt-get update to make sure your system is up to date, and you are good to go.

## Install Docker
We will need Docker installed to work properly. Run the following command to quickly and easily install Docker:
```bash
curl -fsSL test.docker.com -o get-docker.sh && sh get-docker.sh
sudo usermod -aG docker $USER
sudo usermod -aG docker jenkins
```
The second command adds your user to the docker group to enable it to be used without sudo. The third command adds the jenkins user to the docker group so it also has permissions to run docker without sudo. After running, log out and log back into your machine for it to properly take effect. After logging back in run a simple docker command to make sure it is working correctly:
```bash
docker run hello-world
```


## Set up Jenkins
First, install java:
```bash
sudo apt install openjdk-8-jre
```

Then install the Jenkins server itself:
```bash
wget -q -O - https://pkg.jenkins.io/debian/jenkins.io.key | sudo apt-key add -
sudo sh -c 'echo deb http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
sudo apt-get update
sudo apt-get install Jenkins
```

You can then navigate to the Jenkins server using a web-browser at port 8080. If running locally, navigate to ->
- localhost:8080
If running in a cloud machine, as I am, navigate to ->
- <cloud-ip-address>:8080
  
## Log into Jenkins and install plugins
After getting to the Jenkins homepage it should display a page asking to 'Unlock Jenkins' the first time. Access the secret password by catting the secret file created on install:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
Log in, and install the recommended plugins. Then create your admin user (make sure to remember your password!). 

Optionally, As an easier way to view the Jenkins CI pipeline visually you can install the Blue Ocean plugin which I find useful. To do so follow these steps:
- Navigate to ‘Manage Jenkins’ > ‘Plugin Manager’ via the left navigation bars or go to the url: localhost:8080/pluginManager.
- Go to the ‘Available’ tab and type in Blue Ocean. Select the top option, it will install dependencies automatically.
- Click ‘install without restart’.
- Let it install, then refresh the page upon completion and navigate back to localhost:8080. A new option on the left called ‘Blue Ocean’ should be present.

## Install Arm tools 
With Jenkins and Docker squared away let's turn to installing the primary tools from Arm in this CI example: Arm Fast Models (FMs) and Arm Compiler 6 (AC6). It is not really 'installing' in the traditional sense as we will just be downloading the tarballs for AC6 and FMs. Placing the tarballs next to the Dockerfiles that need them will enable the Dockerfile to install it in its own image, handeling all the installation and setup itself.

### Obtain Arm Compiler 6
You can download the most recent Arm Compiler 6 from the Arm website:
https://developer.arm.com/tools-and-software/embedded/arm-compiler/downloads/version-6
Grab the Linx version. This should download a tarball named:
DS500-BN-00026-r5p0-16rel0.tgz
Note you do NOT need to untar the package; Docker will automatically handle that in its environment. 

Get that tarball into your machine if in the cloud, then place in the correct location. The docker container charged with builds will need the compiler, so place this tarball in this specific directory so the Build Dockerfile can find it:
/ci-example/docker-environments/BUILD-bare-metal-env/

Then verify you can build the dockerfile by running this build command:
```bash
docker build --build-arg AC_DIR=./ --build-arg AC_INSTALL=DS500-BN-00026-r5p0-16rel0.tgz -t build-bare-metal-env:latest .
```
After this build, if you run ```docker images``` you will be able to see the 'build-bare-metal-env:latest' image availible with AC6 compiled inside it.

* Note: You must build the docker container manually as Jenkins performs a 'docker pull'. This method is great for more robust set ups where the machine with Jenkins running is connected to a centeral hub where Dockerfiles of maintained building/testing environments are located. However, for simple systems like this example you need to rebuild the Dockerfiles manually outside Jenkins on your machine upon a change in the image you want to propogate to your CI run environments. You can explore giving Jenkins a Dockerfile instead of a Docker image to see if that works better for your setup.

### Obtain Arm Fast Models
You can download the most recent Arm Fast Models package from the Arm website (you may need to signin/create an Arm Developer account to download the tarball):
https://developer.arm.com/tools-and-software/simulation-models/fast-models
Grab the Linx version labeled 'Fast Models 11.10 Evaluation Package'. This should download a tarball named:
FastModels_11-10-022_Linux64.tgz
As before, you do NOT need to untar the package; Docker will automatically handle that in its environment. 

Again, place this tarball in this specific directory so the Test Dockerfile can find it:
/ci-example/docker-environments/TEST-base-metal-env/

Following the same proceedure, download the specific Fast Models system based off the Cortex-M33 and place it in the same directory. You can download it here:
https://developer.arm.com/tools-and-software/simulation-models/fixed-virtual-platforms
and downlaod the 'Cortex-M33 FVP' Linux version.
FVP_MPS2_Cortex-M33_11.10_22.tgz

Place that tgz in the same directory:
/ci-example/docker-environments/TEST-base-metal-env/


Then verify you can build the dockerfile by running this build command:
```bash
docker build -t test-bare-metal-env:latest .
```
After this build, if you run ```docker images``` you will be able to see the 'test-bare-metal-env:latest' image availible with FMs compiled inside it.

* Note: Same with this docker image as above; you must build the docker container manually before running with Jenkins for Jenkins to identify it properly.

### Obtain Arm Licenses
To obtain trail license for both tools please e-mail license.support@arm.com and request a 30 day trial license for Arm Compiler 6 (standalone) and Fast Models, specifing you are using the Cortex-M33 FVP and need both an FVP license and a full Fast Models license (to take advantage of scripting capabilities in the product used in this CI example). The response will be a Serial Number which allows you to create your own license by following the steps outlined in the 'Product License' section on this GitHub page from Arm: https://github.com/ARM-software/Tool-Solutions/tree/master/hello-world_fast-models


## Build Docker environemnts
If you followed the steps above you will have already built the environments for:
- Building the bare-metal software
- Testing the bare-metal software

We need to build the environment for Testing the Linux-based software in python. Luckily everything needed is self-contained in a Dockerfile, so simply navigate to this directory and build the dockerfile:
```bash
cd ci-example/docker-environments/TEST-linux-env/
docker build -t test-linux-env:latest .
```

Upon completion, running a ```docker images``` command should output at least 3 images:
```bash
REPOSITORY                                                 TAG                 IMAGE ID            CREATED             SIZE
test-linux-env                                             latest              0552797b0a8e        2 minutes ago       424MB
test-bare-metal-env                                        latest              9e4a8f178966        9 minutes ago       2.13GB
build-bare-metal-env                                       latest              a4a9a937b1b2        37 minutes ago      1.84GB
```

## Run Pipeline
With your environment initalized with the proper Docker images, Arm tools and Jenkins, the time has come to run a job yourself. The first step in doing so is to click 'Open Blue Ocean' on the main Jenkins page (localhost:8080) on the left-hand side. It will instantly prompt you create a new pipeline. Click on it to proceed.

### Creating a new Pipeline
You already have a pipeline in code predefined, given for free as part of this Git repository. As Jenkins is all about maintaining close in sync with your source code, you will need to connect Jenkins to a source code repository that you have read/write permissions to. The easiest way to get up and running is to fork this repository into your own account and play with it to your heart's content.

If you fork this repo, you can then easily create an ssh key to it via Blue Ocean. Click on 'GitHub' in the Blue Ocean GUI and you'll be prompted to choose one repository under your account. You'll be asked to create an ssh key and will be given a link to do so on your GitHub account. Once this is done on your GitHub account you can copy the ssh key into Jenkins Blue Ocean and proceed by clicking 'Create Pipeline'

### Explore the Example Pipeline
Having gotten this far you should explore the existing example pipeline provided. The Blue Ocean GUI displays visually what is in the Jenkinsfile in this repository. There are two steps, corrosponding to 2 example IoT companies (one creating a bare-metal application based on C and one creating an embedded-linux based application based on Python). 

The first stage is the build, which builds the bare-metal software using Arm Compiler 6. The embedded linux example does not need to be built as python is an interpreted language, and the embedded linux OS itself can be approximated well by the Linux OS in the cloud machine (this is an approximation, but a fair one for certain unit testing needs).

The second stage are the tests. The bare-metal example runs an Arm Fast Model system with a Cortex-M33 and various peripherals via a python script. The bare-metal example takes the built app from the build stage and uses it during the run stage. The linux example runs a python application in a Docker image via pytest. These tests both output results in a standard format in a result.xml file for Jenkins to read and interpret. 

### Run the Pipeline
In the Blue Ocean GUI, click on the 'Save' button in the top right. This will kick off your pipeline running and will save any changes you made to the pipeline for that run. Note that the Jenkins environment will pull the latest source code from the repository you linked to it, and will leverage the Docker images built on the Linux machine to perform builds and tests in an isolated environment.

The run should take under a minute and should result in a yellow ! status. That indicates all steps in the pipeline ran without error but not all tests (reported by the result.xml files) passed. If you click into that run and click the 'Tests' tab in the top right of the screen you can see exactly what tests failed, were skipped, and passed. Note this was done on purpose to display what it looks like to have skipped and failed tests.


## Expand!
Experiment with the various code building blocks here if you are curious about how to change things. Move things around, create new tests, or even add your own software to see how it runs on Arm Fast Models or builds with Arm Compiler 6.

If you have any questions abou the CI example process, Arm tools, etc please send me a message or email at zach.lasiuk@arm.com
