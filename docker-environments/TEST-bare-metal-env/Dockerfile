# # Build command
# #     docker build -t test-bare-metal-env:latest .
#
# # Run command
# #     docker run -it test-bare-metal-env:latest




FROM ubuntu:16.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update -y

### System Functionality ###
RUN apt-get install -y apt-utils iputils-ping python2.7
RUN apt-get install -y python-pip

## Fast Models dependencies
RUN apt-get install -y --fix-missing \
	lsb-core     \
	libxext6     \
	libsm6       \
	libxcursor1  \
	libxft2      \
	libxrandr2   \
	libxt6       \
	libxinerama1



### Tool version selection
ARG FM_INSTALL=FastModels_11-7-035_Linux64
ARG FM_DIR=FastModelsTools_11.7



### Install Fast Models

# # Copy Fast Model install file from same directory as this Dockerfile
COPY $FM_INSTALL.tgz /home/

# # install Fast Models, delete artifacts
RUN tar xvzf /home/$FM_INSTALL.tgz            &&\
		cd $FM_INSTALL                &&\
		./setup.sh --i-accept-the-license-agreement --basepath /arm-tools/     &&\
		rm /home/$FM_INSTALL.tgz      &&\
		rm -r /$FM_INSTALL/

# # add setup to /init.sh
RUN echo "set +e; . /arm-tools/$FM_DIR/source_all.sh" >> /init.sh
