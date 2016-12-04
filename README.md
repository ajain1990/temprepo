# FIST: Fault Injection Simulation and Testing ![Awesome](https://github.com/ajain1990/temprepo/blob/master/FIST%20logo.png)

## Introduction
There are many things about the software which only developer knows can fail, and our software is often expected to be able to handle those failures. But how do we test our failure handling code, when it's not easy to make a failure appear in the first place? One way to do it is to perform fault injection.

According to Wikipedia, "fault injection is a technique for improving the coverage of a test by introducing faults in order to test code paths, in particular error handling code paths that might otherwise rarely be followed. It is often used with stress testing and is widely considered to be an important part of developing robust software".

FIST is a framework that we can use to add fault injection to our Starling code. It aims to be easy to use by means of a simple API, with minimal code impact and little runtime overhead when enabled. This means that for enabling FIST, there are modifications which we have to make in our code and build system.

## FIST Acrchitecture
![Awesome] (https://github.com/ajain1990/temprepo/blob/master/FIST%20architecture.PNG)

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

