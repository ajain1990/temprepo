# FIST: Fault Injection Simulation and Testing ![Awesome](https://github.com/ajain1990/temprepo/blob/master/FIST%20logo.png)

## Introduction
There are many things about the software which only developer knows can fail, and our software is often expected to be able to handle those failures. But how do we test our failure handling code, when it's not easy to make a failure appear in the first place? One way to do it is to perform fault injection.

According to Wikipedia, "fault injection is a technique for improving the coverage of a test by introducing faults in order to test code paths, in particular error handling code paths that might otherwise rarely be followed. It is often used with stress testing and is widely considered to be an important part of developing robust software".

FIST is a framework that we can use to add fault injection to our Starling code. It aims to be easy to use by means of a simple API, with minimal code impact and little runtime overhead when enabled. This means that for enabling FIST, there are modifications which we have to make in our code and build system.

## FIST Acrchitecture
FIST architecture typically consists of the following components:
![Awesome] (https://github.com/ajain1990/temprepo/blob/master/FIST%20architecture.PNG).

* #### [*FIST BuildTool*]
Tool used during package build. It enables FIST points, which are mentioned inside comment (<aoFISTpoint> .. </aoFISTpoint>) in code.

1. It takes directory in which FIST points need to be enabled.

2. First it tries to make a copy of that folder with the name either given by user or it creates directory with name like <src dir>_aofitenable.

3. Once it made a clone of the directory it iterates thorough each file and tries to locate a pattern mentioned below. If it finds any then it removes the above and below part and uncomment the code mentioned between these two the AOFISTPOINT tag.

4. Once this is done, it gives the files to the CPP tool which expands macros present in fistdef.h file.

* #### [*FIST library*]
Stores different fault types, fault locations, fault times, and appropriate hardware semantics or software structures.  Stores various FIST structures and implements library functions. It also maintains FIST event database.

* #### [*FIST Server*]
It facilitates communication between FIST controller and library.  Server defines set of commands and callback functions for each command. The callback function will be called once command is received. Fist controller could send a message to server and wait for a response.

* #### [*FIST Controller*](https://github.com/Gemini-sys/cns/blob/master/core/host/go/aofistdriver/fistctld/fistctl/README.md)
The 'fistctl' utility can be used to administer FIST events.


### FIST Event Actions
There are list of actions which can be specified, in the form of a program, while adding an event in FIST configuration. The control flow of the actions is implicit. Actions that change the execution status of the event generally cause processing to start again at the beginning of the action list when the event hit again. Some actions will result in the current action to proceed to the next in the list. Once the end of the list is reached, we start over again during the next trigger of event.

Currently, there are 4 supported actions namely:

* #### skip
This action can be used to set up an event that only happens once in every n times.
It takes a count argument which is decremented every time the event is triggered until the count
reaches to zero. After this, control will be passed to the next action in the program. If the count is still positive, however, then a break will occur in the action processing until the event is hit next time.

* ### delay
This action causes the event to sleep till the specified time provided in the argument by user. It will introduce a delay point in the code. It is useful when user want to delay processing on an event(like IO read/write, other hsctl operations etc)

* ### stop
This action causes the current event to become disabled. Generally it should be the last instruction in the program provided in list of actions because once control executed this action no further action will get chance to execute. Event need to be enabled explicitly with the help of FIST controller to the start over. 

* ### ioerr
This action causes the current event to generate an IO failure. It should be used with events having type DEVIO/SSDLOG. The error value will be set in the error variable passed in as an argument by
the trigger. If user is not specified any error value then it fails IO with default error.

