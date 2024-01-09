+++
title = 'The importance of generating Test Coverage Reports and some special use cases'
date = 2024-01-09T15:35:58+02:00
draft = false
showtoc = true
tocopen = false
tags = ['testing','coverage','manual-testing','debugging']

+++


## What is a test coverage report?

 A test coverage report provides information about the extent to which a software application has been tested. It measures the code coverage achieved by a set of test cases, indicating which parts of the code have been executed during testing and which parts remain untested.

Test coverage is often expressed as a percentage and can refer to various aspects of  the code, such as statements, branches, functions, etc. Here are some common types of test coverage:

- Statement Coverage: Measures the percentage of code statements that have been executed at least once during testing.

- Branch Coverage: For each decision point (a condition in a decision or repetition structure), it examines whether the true and false outcome have been executed at least once during testing.

- Function Coverage: Examines whether all functions or methods in the code have been invoked at least once during testing.

Why is generating coverage reports important?

*Note:* The examples below are based on JavaScript and they do not necessarily represent good practices. 

## Measuring the coverage of your tests

The most obvious answer is to get some measurement of the coverage of your tests. The goal is to achieve the highest coverage with the minimum number of test cases. Each test should be testing something different than the rest of the tests. A different statement, branch or function that has not been tested yet. This corresponds to some aspect of the requirements that have not been tested yet.

Increasing the test coverage percentages increases the confidence that the code is correct and the requirements have been correctly implemented.

**But there is value to coverage reports beyond the percentages!**

## Analyzing not covered code 

There are other important advantages in generating test coverage reports. In my opinion, it is not so important to see what has been covered but instead it is of extreme usefulness to examine what has **NOT** been covered.

Analysis of uncovered code can provide some useful information about your production and testing code.

- *Is your production code reachable?* --- You may identify some cases in which the production code is not reachable at all. The conditions for reaching this piece of code are never attainable. You can usually safely remove this code.
- *Are there redundant conditions in the production code?* --- In the following example, the first condition is not necessary since if the array is empty will never include any data.

```
someArray.length == 0 || !someArray.includes(someValue)
``` 
- *Are your tests correct?* --- If you are certain that a specific test covers some code and your coverage report shows this piece of code is not covered, then your test is wrong and you need to fix it till the uncovered part of your code is covered. In the following example,  the intention was to check for undefined. Unfortunately, in the test setup, `userId` was defined with the value 0. Unfortunately 0 for JavaScript makes the condition become false and the code is not reached.

```
if (userId) {...}
``` 

## Debugging via coverage

Generation of a  coverage report can replace a debugging activity. Instead of running a debugger in order to track the execution of your code, you can generate a coverage report from the isolated  execution of a single test. 

The coverage report will show you the path your production code took for the specific test. This is something very useful to do when you are creating a new test and you are not sure which path it is already taking, before adding more testing code to your code.

## Manual testing and coverage

Coverage reports are useful for automated testing. What about manual testing?  

You can execute your application and then go through some testing scenarios. Wouldn't you like to know what is the coverage of your manual testing?  

You can measure the coverage of your manual testing the same way you measure the test coverage of your automated tests. 

The example below uses `nyc` for the instrumentation of the code. `nyc` is executed directly against the production application, instead of the tests. `node app.js` starts a backend server for a web application. Perform your usual manual testing process and then stop the server and examine the coverage report.

```
npx nyc --reporter=html --reporter=text node app.js
```

The coverage report will show you all the paths that have been executed from your manual testing process.

This can be very useful especially in legacy applications in which you don't know for sure which code  is executed.

## Closing notes

As you can see, I think that there are many advantages in generating and analyzing test coverage reports.
Most of the above use cases involve examining and analyzing the execution paths instead of just looking at the final percentages.

With the existing automation tools and integrated development environments, generating a coverage report is something that you can easily integrate in your every-day development cycle.

If you can think of other useful cases for test coverage reports, let me know with an email.
