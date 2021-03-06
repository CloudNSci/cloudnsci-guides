# Cloud'N'Sci Refinery Protocol (CNS-RP)

The Cloud'N'Sci Refinery Protocol (CNS-RP) is used to integrate algorithmic data refining solutions into the [Cloud'N'Sci.fi marketplace](http://cloudnsci.fi "The Cloud'N'Sci.fi marketplace"). See the project [README](README.md) file for an introduction to the Algorithms-as-a-Service concept.

In CNS-RP, messages are delivered as tabulator separated lines via standard input/output/error streams (stdin/stdout/stderr). This makes the protocol very simple and easy to implement for various programming languages and operating systems. From now on, we will use \t to indicate the tabulator characters in messages. Because the tabulator character is reserved for this special use, message content part must not include any tabulator characters.

Refinery processes must read incoming messages from the standard input stream (stdin) and write outgoing messages to the standard output stream (stdout). Optionally, refineries can write arbitrary content to the standard error stream (stderr) for logging and debugging purposes. Any output written to stderr will be appended to a refinery specific log file managed by the marketplace. 

## Launching refinery processes   

The marketplace will launch a new instance of a refinery as a stand-alone process whenever needed. This may happen when the first data refining request targeted to the refinery arrives or when another concurrently running instance is needed to handle heavy load. The launch command for the refinery process must be specified by the refinery provider for each offered data refining service type. Thus, providers can create multiple types of data refining services that are based on the same refinery module by giving different launch command parameters. 

When a refinery process is successfully started, it must write the following line to stdout, wherein protocolVersion is the version of the CNS-RP protocol supported by the refinery. 

    HELLO\t<protocolVersion>  

For example, the following HELLO message would indicate that the refinery supports CNS-RP protocol version 1.0.

	HELLO	1.0

The current version of the protocol is 1.0 which is described in the following section. 

## CNS-RP version 1.0

### Initializing

After launching the refinery process and receiving the HELLO message, the marketplace sends a SETUP message which includes an unique refineryId, solution name and version (as submitted to the marketplace) and expected function interface specifications. For each function interface, a function name and the number of input/output/error parameters are given. The first function interface is always named 'main' and it specifies expected parameter counts for this refinery as defined during data refining solution submission (explained in the Solution Provider guide that will be published later). The refinery should read the SETUP message from stdin, check the function interfaces and respond by sending either ALIVE (all OK) or DEAD (something is wrong).

	SETUP\t<refineryId>\tname\tversion\tmain:<in/out/err>\t[<function1:in1/out1/err1> ...]

For example, the following SETUP message would set refineryId=a0000004f, name=big-sort, version=0.1 and specify three function interfaces 'main', 'small-sort' and 'merge'. The main function takes one input data item and generates either one output data or one error data. This refinery can also call function 'small-sort' by giving 1 data item as input and expecting one data item as a result (output or error). The other callable function 'merge' takes 2 input data items.      

	SETUP	a0000004f	big-sort	0.1	main:1/1/1	small-sort:1/1/1	merge:2/1/1

The refinery must check that given function interfaces match the actual implementation. In case everything seems to be OK, the refinery should reply by sending an ALIVE message which must include exactly the same refineryId as given in SETUP and the properties of the refinery as key-value pairs. In this protocol version, there is only one mandatory property 'info' which should briefly describe the functionality of the refinery. In addition, there can be arbitrary number of optional refinery-specific properties such as version number or active setup details that may help in troubleshooting.

	ALIVE\t<refineryId>\tinfo=<description>\t[<key1>=<val1>\t...]

For example:

	ALIVE	a0000004f	info=Sorting algorithm	version=0.1	max-numbers=50000

In case the function interfaces given in SETUP do not match the actual implementation or the refinery is otherwise unable to operate (e.g. invalid configuration, missing resource files), it must send a DEAD message, close the output stream and end the process. Optionally, refinery can also deliver some "last words" explaining the reason for the sudden death.

	DEAD\t<refineryId>\t<lastWords>

For example:

	DEAD	a0000004f	Unknown function 'merge'

### Contexts

Before sending any requests, the marketplace will introduce one or more contexts by sending a CONTEXT message. Each context has an unique ID referred by the following requests and defines a set of context related properties.       

    CONTEXT\t<contextId>\t<contextDir>\t<providerId>\t<applicationId>\t<sessionId>\t<customerId>

For example:

    CONTEXT	900000123	/contexts/900000123	123	456	789	98765

Value of parameter contextDir defines an absolute path to a directory in the local file system that can be used to save context related information which should be remembered between subsequent requests in the same context. Value providerId identifies the provider of the refinery. Value applicationId identifies the applications which made the request. Value sessionId identifies the application session in which the request was made. Value customerId identifies the customer who owns the application session.

Refineries that need context information should maintain a mapping from contextId to the context related data they need. The CONTEXT message can be ignored by stateless refineries which do not utilize context information.  

### Resources

When submitting new refinery releases in the marketplace, providers can optionally identify a list of required resources. The marketplace will deliver available resources to the refinery by sending a RESOURCE message.

    RESOURCE\t<contextId>\t<resourceId>\t<featureId>\t<valueType>\t<value>
    
For example:
 
    RESOURCE	900000123	MyResource	MyFeature1	TXT	MyValue
    RESOURCE	900000123	MyResource	MyFeature2	DIR	/resources/90000123/MyResource/MyFeature2

Value of parameter contextId identifies the context in which the resource is available. Parameters <resourceId> and <featureId> identify the resource feature. Value of parameter valueType can be either TXT or DIR. In case of TXT, parameter <value> defines the value of the feature (e.g. a number or string). In case of DIR, the feature value defines a path in the local file system directory containing the resource value (e.g. an image). 

Refineries that have required resources should maintain a mapping from the contextId to the related resources so that resource values can be found when needed in request execution. Refineries without required resources can ignore RESOURCE messages.  
       
### Request handling

After the initialization phase and context definitions, refinery should wait for data refining requests from stdin. The marketplace will make requests by sending a PERFORM message which includes an unique requestId, an unique contextId and absolute file system paths to a working directory and 1-N input directories wherein N matches the input count defined for the 'main' function interface (see SETUP). The refinery must use the given requestId for all replies related to this request. The contextId refers to a context defined earlier that can be utilized for performing the request.    

	PERFORM\t<requestId>\t<contextId>\t<workDir>\t<inputDir1>\t[<inputDir2> ...]

For example:

	PERFORM	b00000082		900000123	/work	/input1

The refinery should read the input data from specified directories and start processing it. In case the request was completed succesfully, the refinery should deliver the results to the marketplace by sending a READY message including the original requestId and 1-M output directories. Each output directory must be a subdirectory to the working directory given in PERFORM (/work in the example above), but can be named freely. The number of returned output directories must match the output count defined for the 'main' function interface (see SETUP).     

	READY\t<requestId>\t<outputDir1>\t[<outputDir2> ...]

For example:

	READY	b00000082	/work/output1

In case the data refining request fails for any reason, the refinery should send a FAILED message which includes the original requestId and 1-F error directories where F matches the error count defined for the 'main' function interface (see SETUP). Each error directory must be a subdirectory to the working directory given in PERFORM. 

	FAILED\t<requestId>\t<errorDir1>\t[<errorDir2> ...]

For example:

	FAILED	b00000082	/work/error1

When executing a request is stopped for any reason (READY, FAILED, ABORTED), the refinery should clean up all temporary resources allocated for the request and reset the refinery state so that the next request can be started from the same initial conditions as the previous ones. In other words, executing a request should not have any side effects to the following requests. In case this is impossible to achieve, the repository should kill itself after every successful request by sending a DEAD message and closing stdout. That will force the marketplace to launch a new repository process for the next request.  

### Aborting requests

The marketplace may ask to abort an on-going request by sending an ABORT message with the requestId. Supporting ABORT is optional for refineries, so the message can be ignored. However, if request aborting is supported, the refinery should stop processing the request as soon as possible and reply with ABORTED when the request has been stopped. 
 
	ABORT\t<requestId>

For example:

	ABORT	b00000082

The ABORTED message must include the requestId which was aborted. After sending an ABORTED message, the refinery must not send any futher messages referring to the aborted requestId.

	ABORTED\t<requestId>

For example:

	ABORTED	b00000082

### Making subrequests

Refineries may have external dependencies to other data refining functions. In that case, the SETUP message will provide a complete list of functions that may be called by the refinery. Dependencies are managed via the marketplace My Business page when submitting data refining solution releases.

Making a subrequest to an external data refining function is perfomed by sending a CALL message. Each subrequest must be associated to an on-going main request and include a callId which is unique in the scope of the main request. A good practice is to use the order number of the subrequest as the callId, but some other kind of unique ids can be used as well. In addition, the CALL message must include the name of the called function and 1-K input data directories wherein K must match the called function interface specification (see SETUP). Each input data directory must be a subdirectory to the main request working directory. 

	CALL\t<requestId>\t<callId>\t<function>\t<subInputDir1>\t[<subInputDir2> ...]

For example:

	CALL	b00000082	1	merge	/work/sub1	/work/sub2

The marketplace will attempt to execute the function call by making a request to the associated refinery. In case the subrequest is somehow invalid (e.g. unknown function name or wrong number of parameters), the marketplace will respond with the INVALID message. In case the subrequest is completed successfully, the marketplace will deliver results with the RESULTS message. If the subrequests fails, the marketplace will deliver error details by usings the ERRORS message.

The RESULTS message includes the same requestId and callId as the original subrequest. In addition, there are 1-L output data directories, wherein L matches the output count of the called function interface. 

	RESULTS\t<requestId>\t<callId>\t<subOutputDir1>\t[<subOutputDir2> ...]

For example:

	RESULTS	b00000082	1	/result1

The ERRORS message is similar to RESULTS but refers to 1-E error data directories, wherein E matches the error count of the called function interface. 

	ERRORS\t<requestId>\t<callId>\t<subOutputDir1>\t[<subOutputDir2> ...]

For example:

	ERRORS	b00000082	1	/error1

In case the marketplace decides to terminate or deny a subrequest for any reason, it will send an EXCEPTION message to the refinery which started the subrequest. This means that there was nothing wrong in the original subrequest or given input data, but the system just cannot process it at the moment. The EXCEPTION message includes the main requestId and a description for logging purposes. The receiving refinery must terminate all processing related to the given requestId immediately and send FAILED.  

	EXCEPTION\t<requestId>\tdescription

For example:

	EXCEPTION	b00000082	Temporal networking problem
	
### Protocol violation

The INVALID message is a general purpose way to respond to a message which violates the CNS-RP protocol or contains invalid content, e.g. refers to unknown requestId. Any protocol party can reply with INVALID if necessary. Typically, the cause for protocol violation is a bug in refinery implementation which and therefore the marketplace may kill the process immediately.     

	INVALID\t<originalMsgType>\t[<originalMsgArg1> ...] 
	
### Monitoring

The marketplace monitors the health of refinery process by sending PING messages periodically. Refineries should respond to PING with a PONG message as soon as possible to indicate that the process is alive and kicking. Ignoring PING may result in termination of the refinery process in case it hasn't given any signs of healthy activity for a long time. This is necessary to recover from situation in which there is a bug in the algorithm implementation that causes a never-ending "busy-loop" consuming a lot of CPU, RAM and/or file storage. The PING messages includes an unique pingId that must be included also to the PONG reply.
 
	PING\t<pingId>

For example:

	PING	345

The PONG message has also a set of key-value pairs which must include at least property sender=<refineryId>. Optionally, the refinery can also return some useful statistics such as requests=<comma-separated list of active requests> and threads=<number of active threads> separated with semi-colon (;). 


	PONG\t<pongId>\tsender=<refineryId>[;requests=<request-list>;threads=<thread-count>]

For example:

	PONG	345	sender=a0000004f;requests=b00000082,b00000099;threads=2   

In the example above, refinery a0000004f reports that it is currently performing two requests b00000082 and b00000099 on separate threads.  

----------
  
Copyright (c) Cloud'N'Sci Ltd 2015, [http://cloudnsci.fi](http://cloudnsci.fi).
