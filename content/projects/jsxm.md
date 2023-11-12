+++
title = 'JSXM'
date = 2023-11-12T19:22:34+02:00
draft = false
tags = ['Model-based-testing', 'testing', 'automated-test-generation']

hidemeta = true
+++

JSXM (http://jsxm.org/) is a model-based testing tool developed in Java. One needs to provide a model specification of a system and the generated test cases reveal all functional inconsistencies in the implementation of the system.

The tool provides the following functionalities:

- Model animation
- Test generation
- Test transformation

## Model animation

Animation of the model means execution of the model or simulation of the modeled software. Interactive or batch animation allow the model designer to validate the specification, i.e. ensure that the correct functionality is modeled.

## Test generation

The generated test cases are in XML format and they are independent of the technology or programming language of the implementation. This means that the implementation under test can be an application any programming language or implementation platform.

## Test transformation

Since the generated test cases are independent of the underlying technology of the implementation under test, the tests cannot be directly used for testing. A Test Transformer is responsible for transforming the general test cases to concrete test cases.

Currently there is a transforment for JUnit testing.

## Model specification

JSXM models are a special type of extended finite state machines, called Stream X-Machines (SXMs). SXMs allow the description of both the control and the data of a system. The most important advantage of SXMs is their testing method. It can be guaranteed that under certain desing-for-test conditions, the generated test cases reveal all functional inconsistencies in the implementation.

JSXM’s modelling language is based on XML and inline Java code. It supports XSD types. It supports the automatic generation of JUnit test cases for testing Java applications.

An example of a JSXM model of a switch with a counter:

```xml
  <states>
    <state name="OFF" />
    <state name="ON" />
  </states>

  <initialState state="OFF"/>

  <transitions>
    <transition from="OFF" function="SwitchOn" to="ON" />
    <transition from="ON" function="SwitchOff" to="OFF" />
  </transitions>

<memory>
  <declaration>
    int counter;
  </declaration>
  <initial >
    counter = 0;
  </initial>
  <display >
    "On_Counter :" + counter
  </display>
</memory>

  <inputs>
    <input name="on" />
    <input name="off" />
  </inputs>

  <outputs>
    <output name="onOut" />
    <output name="offOut" />
  </outputs>

  <functions>
  <function name="SwitchOn" input="on" output="onOut" xsi:type="OutputFunction" >
  <precondition>
      counter &lt; 2
  </precondition>
  <effect>
      counter++;
  </effect>
  </function>

  <function name="SwitchOff" input="off" output="offOut" xsi:type="OutputFunction" />
  </functions>
```

## How to use

Visit the http://jsxm.org/jsxm-maven-plugin/ site and start by reading the following guides:

- [Installation guidelines](http://jsxm.org/jsxm-maven-plugin/guide/installation.html)
- [Getting started](http://jsxm.org/jsxm-maven-plugin/guide/start.html)
- [Encancing the specification](http://jsxm.org/jsxm-maven-plugin/guide/start_2nd.html)

## Maven plugin for JSXM

JSXM-maven-plugin serves as an interface for the JSXM tool. It provides 13 different goals each bound to a different phase. It supports all the functionality exposed by JSXM (animation, compilation, transformation, test case generation) along with new functionality such as XML validation, Repast agents compilation, graphical interface animation, batch animation, and sample creation. Last but not least, the plugin offers some extra functionality in order to smooth the integration of JSXM to existing Maven projects. For more information visit the [Plugin Documentation](http://jsxm.org/jsxm-maven-plugin/plugin-info.html#/Plugin_Documentation).
