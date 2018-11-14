FROM hyperledger/fabric-ccenv
MAINTAINER Michael Oswald <michael.oswald@trivadis.com>
RUN apt-get -y purge openjdk-8-jre \
 && apt-get update \
 && apt-get -y install openjdk-9-jre \
 && apt-get -y autoremove