+++
title = 'Define your Fluent Interface (DSL) to improve the readability of your tests'
date = 2023-12-20T14:06:46+02:00
draft = false
showtoc = true
tocopen = false
tags = ['fluent-interface','DSL','domain-specific-language','method-chaining','testing']
+++


Although everybody agrees that automated testing is essential for achieving high software quality, writing tests cases is not always easy. Especially, setting up the test scenarios that involve the initialization of many objects can be difficult. I recently started using the concepts of fluent interfaces and  DSL languages for writing my tests in such cases. Using this approach, tests become easier to write and more importantly much easier to read understand.

## What is a DSL

A Domain Specific Language (DSL) is a programming language for a specific domain (in contrast to general purpose programming languages). DSLs can be internal or external. An external DSL needs its own parser and compiler or interpreted to be executed. An internal DSL is hosted within an existing general purpose programming language, so there is no need to write any extra tools.

Writing a program in the style of DSL increases the readability of your program. In this post, I will use an example of writing test cases for an application in Java.

## Testing a Network Diagram application

Some time ago, I created a network diagram application for project management https://github.com/dranidis/network-diagram. The goal of the application is to read tasks from a file,  create a network diagram,  and calculate the critical paths. I recently revisited the application and started doing some enhancements. In order to refresh my memory about the application I had written long ago, I took a look at the tests I had written and I found them quite difficult to read.

Let me give you an example of a test:

```java
@Test
public void criticalPath_Should_....() {
    List<TaskData> taskList = new ArrayList<>();
    taskList.add(new TaskData("A", 4,
            Arrays.asList(new DependencyData[] {})));
    taskList.add(new TaskData("B", 4,
            Arrays.asList(new DependencyData[] {})));
    taskList.add(new TaskData("C", 2,
            Arrays.asList(new DependencyData[] {
              new DependencyData("A", "FS"),
              new DependencyData("B", "FS") })));
    taskList.add(new TaskData("D", 2,
            Arrays.asList(new DependencyData[] {
              new DependencyData("A", "FS"),
              new DependencyData("B", "FS") })));
    taskList.add(new TaskData("E", 3,
            Arrays.asList(new DependencyData[] {
              new DependencyData("A", "FS"),
              new DependencyData("B", "SF"),
              new DependencyData("C", "SS"),
              new DependencyData("D", "FS") })));
    taskList.add(new TaskData("F", 3,
            Arrays.asList(new DependencyData[] {
              new DependencyData("A", "FS"),
              new DependencyData("B", "FS"),
              new DependencyData("C", "FS"),
              new DependencyData("D", "FF") })));
...
}
```

At the given snippet, I have skipped the assertions. The first part of the test focuses on setting up the test: creating all the necessary objects and their connections. In this case, the goal is to create a network diagram object with several task objects each having an id, a duration and a list of dependency objects to other tasks. A dependency has an id (a reference to the task it depends to) and the type of dependency ("FS" indicates a Finish to Start dependency).

Employing techniques such as fluent interface and  method chaining, this piece of code may look like this:

```java
@Test
public void criticalPath_Should_WorkWithTransitiveDependencies() {
    TaskDataList tasklist = taskList()
            .add(task("A", 4))
            .add(task("B", 4))
            .add(task("C", 2).withPred("A").withPred("B"))
            .add(task("D", 2).withPred("A").withPred("B"))
            .add(task("E", 3).withPred("A").withPred("B", "SF")
                             .withPred("C", "SS").withPred("D"))
            .add(task("F", 3).withPred("A").withPred("B")
                             .withPred("C").withPred("D", "FF"));
```

The resulting code is shorter, cleaner, less cluttered and easier to read and understand.

## Creating a fluent interface

A fluent interface is an object-oriented interface whose goal is to increase code readability by creating a (small and simple) DSL and its design is based on method chaining. A fluent interface API is designed to be readable and flow.  The construction of such an API involves more effort in its implementation. It also might make debugging of the code more difficult. So there is a price to pay for readability.

Let's take a closer look at the details and the differences between the old code and the new fluent interface:

```java
List<TaskData> taskList = new ArrayList<>();

taskList.add(new TaskData("C", 2,
        Arrays.asList(new DependencyData[] {
          new DependencyData("A", "FS"),
          new DependencyData("B", "FS") })));
```                  
Compare with:

```java
TaskDataList tasklist = taskList()
        .add(task("C", 2).withPred("A").withPred("B"))
```
To achieve this style we have to define the following methods either as part of our test code or as part of our production code (in case we use
the fluent interface in other parts of our program).

The definition of `taskList`:
```java
    public static TaskDataList taskList() {
        return new TaskDataList();
    }
```
The `taskList` method is a public static method that creates a new task list.
It is a typical factory method. Making the method static allows us to use it without an object (even a singleton).
 This method needs to be statically imported with:
`import static .....TaskDataList.taskList;` in order to use it without the class name.

The definition of the `add` method:

```java
    public TaskDataList add(TaskData task) {
        tasks.add(task);
        return this;
    }
```
The method returns the `this` object to allow method chaining. This allows us to put several calls to `add` chained like this:
```java
taskList().add(task("A")).add(task("B"));
```

Notice the argument to the method `add`: they are calls to the `task` method, that create the tasks. The definition of the `task` method:
```java
    public static TaskData task(String id, int duration) {
        return new TaskData(id, duration);
    }
```

It is again a public static factory method.

Finally, the `withPred` method uses method chaining to allow the additions of many dependencies to the task. The definition of 
the `withPred` method:

```java
    public TaskData withPred(String predId) {
        predIds.add(new DependencyData(predId));
        return this;
    }
```
Notice that the method is also responsible to create the `DependencyData` object.

## Caution

Using method chaining and a fluent interface might improve code readability but there are downsides as well: 

- Code written using a fluent interface is very hard to debug. Where do you put the breakpoint? 
- If there is logic involved in the methods of the chain, it is not always clear when the logic is executed. There are many examples of method chaining, where the logic is usually executed at the end, but nothing prevents intermediate methods to do things as well.  I would avoid using method chaining in  cases
where complex logic is involved.
- The signatures of the methods violate usual conventions. Setter methods which by convention have a `void` return type, now return an object. You could still however use the method as usually by ignoring the returned value.


## Links

- https://martinfowler.com/bliki/FluentInterface.html
