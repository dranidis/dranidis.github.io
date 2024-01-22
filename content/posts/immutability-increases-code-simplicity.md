+++
title = 'Immutability increases Code Simplicity'
date = 2024-01-21T15:15:02+02:00
draft = false
tags = ['immutability','value-object','domain-driven-design','simplicity']

+++

![mutation](/posts/mutability.png)

Immutability has many advantages, as many functional programming proponents will argue. Just to mention a few, immutable classes are side effect free and are easily testable. I am not going to go into the details of listing all the advantages of immutability in this short article. I will focus mainly on the effects on code simplicity and code comprehension.

Recently, I revisited one of my older projects and by examining the code, I found that a class could be immutable, or its objects could be value objects using Domain Driven Development (DDD) terminology. According to Martin Fowler, who probably came up first with the term, a value object is:

> A small simple object, like money or a date range, whose equality isn't based on identity.

The equality of two value objects is based on the comparison of their properties. Two value objects with exactly the same property values are equivalent, exactly the same way two 50 Euros banknotes are equivalent (unless you are modelling a banknote processing application where the serial number of the banknote is important).

Back to the story, I decided to change the class to an immutable class by replacing all mutator methods with methods that instead return new instances of the class. Of course, I had to refactor the rest of the code that used the old mutator methods first. I was fortunate to have an extensive set of tests that made me feel very secure about the changes.

The project was a project I mentioned in another one of my posts: [Define your Fluent Interface (DSL) to improve the readability of your tests](/posts/define-your-dsl-for-tests/). It is a network diagram application for project management https://github.com/dranidis/network-diagram.

## The mutable Path class

The `Path` class represents a sequence of tasks in the network diagram. It is implemented internally with a list of `Task` objects.  I am listing just the mutator methods:

```
public class Path {

  private List<Task> tasks;
...

  public void addTask(Task task) {
    tasks.add(task);
  }

  public void removeLastTask() {
    this.tasks.remove(this.tasks.size() - 1);
  }
}
```
Note: Yes, I know there should be a check for an empty list in the `removeLastTask`! Consider it as a precondition.

There is no reason for this class to be mutable.
Its instances are clearly value objects. There is no identity involved and two paths having exactly the same sequence of tasks are considered  equivalent for the purpose of the application.


## The immutable Path class

The two methods `addTask` and `removeLastTask` are the methods which mutate the objects of the class and need to be replaced.
I am listing just the replaced methods:
```
...
  public Path addTask(Task task) {
    List<Task> newTasks = new ArrayList<>(this.tasks);
    newTasks.add(task);
    return new Path(newTasks);
  }

  public Path removeLastTask() {
    List<Task> newTasks = new ArrayList<>(this.tasks);
    newTasks.remove(this.tasks.size() - 1);
    return new Path(newTasks);
  }
...
```

The new methods, instead of changing the properties of the caller object, return new instances of the class with modified properties.

## Refactoring the clients of the `Path` class

The two replaced methods were called by two methods of another class. The first one was quite easy to change:
```
private void addTaskToPaths(Task task, List<Path> paths) {
  boolean added = false;
  for (Path path : new ArrayList<>(paths)) {
    List<Task> predecessorTasks = getPredecessorTasksInPath(task, path);
    if (!predecessorTasks.isEmpty()) {
      appendTaskToPaths(task, predecessorTasks, path, paths);
      added = true;
    }
  }
  if (!added) {
    Path path = new Path();
    path.addTask(task);
    paths.add(path);
  }
}
```
The mutation via `addTask` was performed on an empty path, inside the last `if` statement. Replacing the three lines with one already made the code simpler:
```
  if (!added) {
    paths.add(new Path().addTask(task));
  }
```

The second client method was not so easy to change. I had trouble understanding what my intention was when I wrote the original code!

Here is the method:

```
private void appendTaskToPaths(Task task, List<Task> predecessorTasks, Path path, 
                                List<Path> paths) {
  if (predecessorTasks.contains(path.lastTask())) {
      path.addTask(task);
  } else {
      Path clonedPath = new Path(path.tasks());
      clonedPath.removeLastTask();
      clonedPath.addTask(task);
      paths.add(clonedPath);
  }
}
```

The method `appendTaskToPaths` receives a `task` to be "appended" to the paths (according to the method's name), a list of tasks `predecessorTasks` the tasks depends on, a `path` (whose presence was not clear to me), and a list of paths `paths`. 
The "then" part of the `if` statement changes only the `path` argument. The "else" part of the `if` statement changes only the `paths` argument. How is this method supposed to "append" (in some way) the task to the `paths`, if in the "then" part it only appends the `task` to the `path`?
I had to examine the previous method `addTaskToPaths` which was the only one calling `appendTaskToPaths`. 

**Can you figure out how does the method work? How does it always manage to change the `paths`? If you can, congratulations!**



For me, the author of the code, the intention was not clear and I spent quite a long time to figure out what is happening. That's bad news for code simplicity and it is a clear sign of **accidental complexity**.

> Accidental complexity relates to problems which engineers **create** and can fix. 

I created this mess and I **HAD** to fix it.



## Aha!

As it turned out, the `path` argument is one of the elements in the `paths` list. The method worked because the `Path` class was mutable. The elements of the list could be changed through their references in the list. The "then" part mutated an existing path in the list, whereas the "else" part added a new path to the list.

Once I understood what was happening, I rewrote the method using the new immutable `Path`: 
```
private void appendTaskToPaths(Task task, List<Task> predecessorTasks, Path path, 
                                List<Path> paths) {
  if (predecessorTasks.contains(path.lastTask())) {
    paths.remove(path);
    paths.add(path.addTask(task));
  } else {
    paths.add(path.removeLastTask().addTask(task));
  }
}
```

I think that the intention of this method is now more clear and explicit. It changes the `paths` list by either replacing an existing path (by removing the old and adding the new path), or by adding a new path (where the last task is removed and the new task is added). The role of the `path` parameter also became more clear. It is explicit that the `path` is part of the list `paths`. Additionally, there is no need for cloning, since the class is immutable and new instances are returned anyway.

**Note**: Look for cases in your code where you need to clone objects. They are candidates for value objects!

I am pretty sure (wishful thinking), that in the future I will be able to understand faster what the method does. Probably, it would be a good idea to rename the method as well, since it is clearly not a simple append.

As a bonus, making the intent of the refactored method explicit, allowed me to easily identify a bug in the implementation, which I will not discuss in this article. 

## Conclusions

As an afterthought, I would have avoided all this trouble, if I had modelled the `Path` class as immutable from the beginning. In that case, it would have never passed my mind to implement the `appendTaskToPaths` the way it was. It would simply not be possible! Restricting myself to using paths as immutable value objects would have made the intent of the code clear from the beginning.

Does immutability always increase code simplicity? Sometimes, it might be simply easier to mutate the state of an object instead of creating a new instance. It might also be a matter of preference. In case of uncertainty, trying both approaches might reveal what's best. If however you are in doubt, I would strongly suggest to go the immutability path. There are less chances you will regret it. I did!

Immutability is certainly not a **silver bullet** (are there any?) but the more you use it, the more you will embrace it.


Thanks for reading! Let me know your thoughts!

## Links

https://martinfowler.com/eaaCatalog/valueObject.html
https://en.wikipedia.org/wiki/No_Silver_Bullet
