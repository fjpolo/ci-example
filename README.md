# ci-example

## Machine Setup
Tested on a new Linux machine, Ubuntu 18.04. Do a apt-get update to make sure your system is up to date:
- sudo apt-get update

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






