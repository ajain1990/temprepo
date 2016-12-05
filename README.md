# FIST: Fault Injection Simulation and Testing ![](https://github.com/ajain1990/temprepo/blob/master/FIST%20logo.png)

## Introduction
There are many things about the software which only developer knows can fail, and our software is often expected to be able to handle those failures. But how do we test our failure handling code, when it's not easy to make a failure appear in the first place? One way to do it is to perform fault injection.

According to Wikipedia, "fault injection is a technique for improving the coverage of a test by introducing faults in order to test code paths, in particular error handling code paths that might otherwise rarely be followed. It is often used with stress testing and is widely considered to be an important part of developing robust software".

FIST is a framework that we can use to add fault injection to our Starling code. It aims to be easy to use by means of a simple API, with minimal code impact and little runtime overhead when enabled. This means that for enabling FIST, there are modifications which we have to make in our code and build system.

## FIST Architecture
FIST architecture typically consists of the following components:
![] (https://github.com/ajain1990/temprepo/blob/master/FIST%20Architecture.PNG)

#### FIST BuildTool
It is the initial and the crucial phase where all the FIST points which are mentioned inside comment (\<aoFISTpoint\> .. \</aoFISTpoint\>) gets converted into code.

It takes the directory in which FIST points need to be enabled, then it copies the directory with the name either suggested by user or it would be named as  \<src dir\>_aofistenable. Once the directory is copied, then it switches to that directory and iterates through each file and tries to examine the pattern briefed below.

```
/* <aoFISTPoint>
 * FIST_TRIGGER_RETURN(“eventXYZ”, 1, “Operation failed due to FIST point”)
 * </aoFISTPoint> */
```

If it finds the pattern, then first it eliminates the beginning (\<aoFISTPoint\>) and ending (\</aoFISTPoint\>) tag and uncomments the code written between them. Once this is done it hands over the file to the CPP tool which would further expand the FIST macros by taking the definitions from the fistdef.h file.

#### FIST Library
It is composed of different FIST structures which includes event definitions, various actions and their attributes, it maintains FIST events database in the form of \<key-value\> pair. It also implements different functions for interacting with event DB, concurrent queries to the event DB are synchronized by mutex lock.

FIST Controller communicates with the server to invoke different operations, which in turns calls the corresponding library functions to accomplish the task by updating the config database. Here the Operation would equate to add/remove/enable/disable event etc.

While performing any task in the code if FIST API’s gets encountered, then unique event identifier (specified with  API) would be retrieved and examined in the config db with the help of functions exposed by library and if the event is found then the corresponding actions to it would be triggered. 

#### FIST Server
It facilitates communication between FIST controller and library.  Server defines set of commands and callback functions for each command. The callback function will be called once command is received. Fist controller could send a message to server and wait for a response.

#### FIST Controller
The 'fistctl' utility is used to administer FIST events.

[*Read more FIST Controller*](https://github.com/Gemini-sys/cns/blob/master/core/host/go/aofistdriver/fistctld/fistctl/README.md)

## How to use FIST Framework
FIST in Starling project can be used by inserting points of failure, using different macros defined in fistdef.h file. It is recommended to use meaningful names for points of failure, to easily identify their purpose. 

Usually, we don’t want to enable FIST points in non-debug build. To achieve this, FIST macros are commented out. This would make sure that non-debug builds won’t have a single trace of fault injection code, but with the help of FIST build tool it would be easy to create a fist enabled binary, for testing purposes.

## FIST Instrumentations
FIST instrumentations allow us to test code path which would not normally be exercised in everyday use. These can be inserted into the relevant code. These are not enabled by default, FIST controller is used to enable them. Whenever this instrumentations hit, first it determines whether the associated event is added by user or not. If yes then related action will be taken else there will be no change in behavior of code flow. 

A summary of the appearance and use of instrumentation macro is shown below.

##### FIST_IMPORT_PACKAGE()
This macro is used to import all required packages for enabling FIST instrumentation in a particular golang file. If we want to put any FIST points in the file then this macro has to be written in the start of file.

##### FIST_START_SERVER()
This macro captains set of instructions to start FIST server. 

##### FIST_TRIGGER_RETURN(“eventName”,  retArg1, retArg2, …)
This macro forces function to return with provided return argument(s), if associated event is present in FIST configuration. Event name is the unique identifier that is used to scan the all loaded events.

```
/* <aoFISTPoint>
 * FIST_TRIGGER_RETURN(“eventXYZ”, 1, “Operation failed due to FIST point”)
 * </aoFISTPoint> */
```

##### FIST_TRIGGER_ACTION (“eventName”, instruction1; intruction2; …)
This macro is used to trigger instructions if the associated event is enabled by the user. Instructions can be any valid golang expressions or statement. Multiple actions should be comma separated with each other.

```
/* <aoFISTPoint>
 * FIST_TRIGGER_ACTION(“eventXYZ”, a = 1; b = 2; c = 3;)
 * </aoFISTPoint> */
```	

##### FIST_TRIGGER_DEVIO_EVENT (interface, callbackFunc) & FIST_TRIGGER_SSDLOG_EVENT(dev, offset, len, callbackFunc, interface) 
These macros are used to fail IO/SSDLOG on a particular device(or in general if user not specified an device). As we have seen earlier the event name is provided with all above mentioned macros which will be later searched in FIST configuration for triggering respective event.  But with these two macros are nameless. Both of these checks all loaded failure triggers like device, offset etc to cause an associated action trigger. The action can be anything like failing IO/SSDLOG on specified device or additional delay can be added also.

## FIST Event Actions
There are list of actions which can be specified, in the form of a program, while adding an event in FIST configuration. The control flow of the actions is implicit. Actions that change the execution status of the event generally cause processing to start again at the beginning of the action list when the event hit again. Some actions will result in the current action to proceed to the next in the list. Once the end of the list is reached, we start over again during the next trigger of event.

Currently, there are 4 supported actions namely:

* #### skip
This action can be used to set up an event that only happens once in every n times.
It takes a count argument which is decremented every time the event is triggered until the count
reaches to zero. After this, control will be passed to the next action in the program. If the count is still positive, however, then a break will occur in the action processing until the event is hit next time.

* #### delay
This action causes the event to sleep till the specified time provided in the argument by user. It will introduce a delay point in the code. It is useful when user want to delay processing on an event(like IO read/write, other hsctl operations etc)

* #### stop
This action causes the current event to become disabled. Generally it should be the last instruction in the program provided in list of actions because once control executed this action no further action will get chance to execute. Event need to be enabled explicitly with the help of FIST controller to the start over. 

* #### ioerr
This action causes the current event to generate an IO failure. It should be used with events having type DEVIO/SSDLOG. The error value will be set in the error variable passed in as an argument by
the trigger. If user is not specified any error value then it fails IO with default error.

