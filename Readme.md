# Hyperledger and Java
@(hyperledger)
## Implemnting Chaincode in Java
Hyperledger Fabric now supports Java chaincode since [version 1.3](https://jira.hyperledger.org/browse/FAB-8063).  Implementing and running chaincode in java is still a bit cumersome and not straight forward. Nevertheless I started to explore the new functionality and succeeded in running and invoking chaincode written in Java.
Besides writing chaincode in Java, there is also great support from [Hyperledger Fabric Java SDK](https://github.com/hyperledger/fabric-sdk-java) for accessing chaincode (written in any supported language) from Java.
My journey here is to describe a way to implement both in Java
- Chaincode
- Client code (via SDK)

This document is an adaption of the official [Chaincode for Developers](https://hyperledger-fabric.readthedocs.io/en/release-1.3/chaincode4ade.html).
So I make use of the development network provided by hyperledger, which is part of the [Hyperledger Fabric Samples](https://github.com/hyperledger/fabric-samples) project.
The development network consists of the following components
- single orderer (no kafka)
- one peer
- Chaincode environment (ccenv) with jre8 installed
- tools for accessing and communicating with the chaincode running on the peer


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

When I started to work with the provided docker containers from development network, I ran into the following issue,
when instantiating the chaincode

```
    Exception in thread "Thread-2" java.lang.NoSuchMethodError: java.nio.ByteBuffer.flip()Ljava/nio/ByteBuffer;
	at org.hyperledger.fabric.shim.impl.ChaincodeStubImpl.computeBinding(ChaincodeStubImpl.java:97)
	at org.hyperledger.fabric.shim.impl.ChaincodeStubImpl.<init>(ChaincodeStubImpl.java:83)
	at org.hyperledger.fabric.shim.impl.Handler.lambda$handleInit$0(Handler.java:248)
	at java.lang.Thread.run(Thread.java:748)
```

This error is due to a breaking change from [JDK8 to JDK9](https://github.com/plasma-umass/doppio/issues/497).
To fix this issue, I created a new docker container which adds jre9 insteadof jre8 and added it to the docker-compose.yaml
of the development network.

### Terminal 1 Run the Network
After the mentioned fix, we can now start our network in terminal 1
```
$ cd network
$ docker-compose -f docker-compose-simple.yaml up
```
This terminal we can leave in the background since we don't need to enter/change anything here.


## Implementing Chaincode in Java
I took the project which is under "fabric-samples/chaincode/chaincode_example02/java" and added the following
lines to the deployment section of the build.gradle script

```
 compile 'ch.qos.logback:logback-classic:1.1.7'
 compile 'org.slf4j:jcl-over-slf4j:1.7.21'
```

This lines ensure that there is a logger implementation and that all logs are centralized and outputted via logback-classic.

Now we are able to build the shadow jar

```
    $ ./gradlew clean shadowJar
    $ ls  build/libs/chaincode.jar
    build/libs/chaincode.jar
```

### Terminal 2: Start the Chaincode
Now we are ready to run our chaincode in the chaincode docker container (containing our new jre9 installed)

```
    $ docker exec -e COLUMNS=151 -it chaincode bash
    $ CORE_PEER_ADDRESS=peer:7052 CORE_CHAINCODE_ID_NAME=mycc:0 java -jar  build/libs/chaincode.jar
```


The java process in the chaincode docker container must be running all the time, while communication with the chaincode.

We execute the *SimpleChaincode::main* method and assign it the *name=mycc* and the *version=0*.

```
 Here is one open point to me:
 why do we put CORE_PEER_ADDRESS=peer:7052?
 In the file chaincode-docker-devmode/docker-compose-simple.yaml
 the peer is configured with CORE_PEER_ADDRESS=7051
 ==> But only this way it is working!
```

### Terminal 3: Install and Instantiate the Chaincode
Now we have the network running and also our chaincode running on on the ccenv docker image.
We need now to  install the chaincode and later instantiate the chaincode before we can interact with it.
#### Install the Chaincode
Now we are ready to install the chaincode on the *cli* docker component.

```
    $ docker exec -e COLUMNS=151 -it cli bash
    $ peer chaincode install -p chaincode/src/ -n mycc -v 0 -l java
    ...
    Installed remotely response:<status:200 payload:"OK" >
```

Here we are installing the chaincode and providing the *path=chaincode/src/*, the *name=mycc*, the *version=0* and the *language=java* to it.

#### Instantiate Chaincode
The very last step to make our chaincode ready is to instantiate ist.

```
    peer chaincode instantiate -n mycc -v 0 -c '{"Args":["init","a","10","b","20"]}' -C myc
```

Here we are instantiating our chaincode with the *name=mycc*, the *version=0*, the *chaincodeID=myc* (this is hardcoded in the script.sh startup script of the *cli* docker container and is part of the pregenerated crypto material in this container) and the constructor arguments: {"Args":["init","a","10","b","20"]}.








