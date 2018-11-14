# Hyperledger and Java
@(hyperledger)
## Implemnting Chaincode in Java
Hyperledger Fabric now supports Java chaincode since [version 1.3](https://jira.hyperledger.org/browse/FAB-8063).  Implementing and running chaincode in java is still a bit cumersome and not straight forward. Nevertheless I started to explore the new functionality and succeeded in running and invoking chaincode written in Java.
Besides writing chaincode in Java, there is also great support from [Hyperledger Fabric Java SDK](https://github.com/hyperledger/fabric-sdk-java) for accessing chaincode (written in any supported language) from Java.
My journey here is to describe a way to implement both in Java
- Chaincode
- Client code

This document is an adaption of the official [Chaincode for Developers](https://hyperledger-fabric.readthedocs.io/en/release-1.3/chaincode4ade.html).  So I make use of the development network provided by hyperledger. I started my experience with the playground of [Hyperledger Fabric Samples](https://github.com/hyperledger/fabric-samples). Here are the main characteristica of this development network

> **Testing Using dev mode**
>
> Normally chaincodes are started and maintained by peer. However in
> “dev mode”, chaincode is built and started by the user. This mode is
> useful during chaincode development phase for rapid
> code/build/run/debug cycle turnaround.
>
> We start “dev mode” by leveraging pre-generated orderer and channel
> artifacts for a sample dev network. As such, the user can immediately
> jump into the process of compiling chaincode and driving calls.

The network (Terminal 1) consists of
- single orderer
- peer
- ccenv (chaincode environmet): Terminal 2
- tools for accessing and communicating with the chaincode: Terminal 3

## Implementing Chaincode in Java
Checkout the source and compile the mentioned Java sample, which is under "fabric-samples/chaincode/chaincode_example02/java".
But before compiling them, you may want to add a logging implementation to the executable jarfile. To do so just add the following lines in the dependencies section of the gradle script

> compile group: 'ch.qos.logback',name:'logback-classic',version: '1.1.7'
> compile group: 'org.slf4j', name: 'jcl-over-slf4j', version: '1.7.21'

After editing the build.gradle file go an build the executable jar

    $ cd chaincode/chaincode_example02/java
    $ gradle clean shadowJar
    $ ls  build/libs/chaincode.jar
    build/libs/chaincode.jar



For running the complete example we need 3 Terminals.
### Terminal 1: Run the Network

    cd chaincode-docker-devmode
    docker-compose -f docker-compose-simple.yaml up

This terminal we can leave in the background since we don't need to enter/change anything here.
### Terminal 2: Start the Chaincode
First we open a new terminal and enter to the *chaincode* docker

    $ docker exec -it chaincode bash
    # java -version
    openjdk version "1.8.0_181"
    OpenJDK Runtime Environment (build 1.8.0_181-8u181-b13-0ubuntu0.16.04.1-b13)
    OpenJDK 64-Bit Server VM (build 25.181-b13, mixed mode)


Here in terminal two, we have an issue, that should be reported to Hyperledger Fabric: When running the docker container provided, we have a a pre installed openjdk 1.8.
Later, when we try to instantiate the chaincode we will run in the issue of

    Exception in thread "Thread-2" java.lang.NoSuchMethodError: java.nio.ByteBuffer.flip()Ljava/nio/ByteBuffer;
	at org.hyperledger.fabric.shim.impl.ChaincodeStubImpl.computeBinding(ChaincodeStubImpl.java:97)
	at org.hyperledger.fabric.shim.impl.ChaincodeStubImpl.<init>(ChaincodeStubImpl.java:83)
	at org.hyperledger.fabric.shim.impl.Handler.lambda$handleInit$0(Handler.java:248)
	at java.lang.Thread.run(Thread.java:748)

This error is due to a breaking change from [JDK8 to JDK9](https://github.com/plasma-umass/doppio/issues/497) .
To fix this issue, we first have to remove jre8 and install jre9 in our *chaincode* docker container. So we go as follows in our  chaincode docker container

    $ docker exec -it chaincode bash
    # sudo apt-get purge openjdk-8-jre
    # apt-get update
    # apt-get install openjdk-9-jre
    # java -version
    openjdk version "9-internal"
    ...

Now we are ready to run our chaincode in the chaincode docker container (containing our new jre9 installed)

    # cd chaincode_example02/java/
    # CORE_PEER_ADDRESS=peer:7052 CORE_CHAINCODE_ID_NAME=mycc:0 java -jar  build/libs/chaincode.jar

The java process in the chaincode docker container must be running all the time, while communication with the chaincode.

We execute the *SimpleChaincode::main* method and assign it the *name=mycc* and the *version=0*.

> Here is one open point to me:
> why do we put CORE_PEER_ADDRESS=peer:7052?
> In the file chaincode-docker-devmode/docker-compose-simple.yaml
> the peer is configured with CORE_PEER_ADDRESS=7051
> ==> But only this way it is working!




### Terminal 3: Install and Instantiate the Chaincode
Now we have the network running and also our chaincode running on on the ccenv docker image.
We need now to  install the chaincode and later instantiate the chaincode before we can interact with it.
#### Install the Chaincode
Now we are ready to install the chaincode on the *cli* docker component.

    $ docker exec -it cli bash
    # peer chaincode install -p chaincode/chaincode_example02/java/src/ -n mycc -v 0 -l java
    ...
    Installed remotely response:<status:200 payload:"OK" >

Here we are installing the chaincode and providing the *path=chaincode/chaincode_example02/java/src/*, the *name=mycc*, the *version=0* and the *language=java* to it.

### Instantiate Chaincode
The very last step to make our chaincode ready is to instantiate ist.

    peer chaincode instantiate -n mycc -v 0 -c '{"Args":["init","a","10","b","20"]}' -C myc

Here we are instantiating our chaincode with the *name=mycc*, the *version=0*, the *chaincodeID=myc* (this is hardcoded in the script.sh startup script of the *cli* docker container and is part of the pregenerated crypto material in this container) and the constructor arguments: {"Args":["init","a","10","b","20"]}.








