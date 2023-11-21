+++
title = 'Define Your DSL for Tests'
date = 2023-11-20T23:36:59+02:00
draft = true
+++

Although everybody agrees that automated testing is essential for achieving high software quality, writing tests cases is not always easy. Especially setting up the test scenario can be difficult. I recently starting using a simple DSL language for setting up my tests. Tests become easier to write and more important much easier to understand.

## What is a DSL

A domain specific language DSL is a programming language for a specific domain (in contrast to general purpose programming languages). DSLs can be internal or external. An external DSL needs its own parser and compiler or interpreted to be executed. An internal DSl is hosted within an existing general purpose programming language, so there is no need to write any extra tools.

Writing a program in the style of DSL increases the readability of your program.

## Testing a Network Diagram application

Some time ago I created a network diagram application for project management. The goal of the application is to read tasks from a file and then create a network diagram and finding the critical paths. I recently revisited the application and starting doing some enhancements. In order to understand the application I took a look at the tests I had written and I found them quite difficult to read.

Let me give you an example of a test:

```java
    @Test
    public void criticalPath_Should_WorkWithTransitiveDependencies_WhenMoreThanOnePath()
            throws DuplicateTaskKeyException, KeyNotFoundException {
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
                  new DependencyData("B", "FS"),
                  new DependencyData("C", "FS"),
                  new DependencyData("D", "FS") })));
        taskList.add(new TaskData("F", 3,
                Arrays.asList(new DependencyData[] {
                  new DependencyData("A", "FS"),
                  new DependencyData("B", "FS"),
                  new DependencyData("C", "FS"),
                  new DependencyData("D", "FS") })));

        NetworkDiagram nd = new NetworkDiagram();
        DiagramNetworkReaderService.processTaskList(nd, taskList);

...

    }
```

I have skipped the assertions. The first part of the test focuses on setting up the test with the necessary object. In this case, the goal is to create a network diagram with several tasks each having an id and a duration and a list of dependencies to other tasks. A dependency has an id (a reference to the task it depends to) and the type of dependency ("FS" indicates a Finish to Start dependency).

Employing techniques called fluent interface, method chaining this piece of code looks like:

```java
    @Test
    public void criticalPath_Should_WorkWithTransitiveDependencies_WhenMoreThanOnePath()
            throws DuplicateTaskKeyException, KeyNotFoundException {
        TaskDataList tasklist = taskList()
                .add(task("A", 4))
                .add(task("B", 4))
                .add(task("C", 2).withPred("A").withPred("B"))
                .add(task("D", 2).withPred("A").withPred("B"))
                .add(task("E", 3).withPred("A", "FS").withPred("B", "FS")
                                  .withPred("C", "FS").withPred("D", "FS"))
                .add(task("F", 3).withPred("A", "FS").withPred("B", "FS")
                                  .withPred("C", "FS").withPred("D", "FS"));
```

The resulting code is shorter, cleaner, less cluttered and easier to read and understand.

A fluent interface is an object-oriented interface whose goal is to increase code readability by creating a (small) DSL and its design is based on method chaining.

## Links

- https://martinfowler.com/bliki/FluentInterface.html
