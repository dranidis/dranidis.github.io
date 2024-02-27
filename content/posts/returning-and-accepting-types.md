+++
title = 'Returning and Accepting Types'
date = 2024-02-03T02:15:20+02:00
draft = true
+++

https://enterprisecraftsmanship.com/posts/return-the-most-specific-type/

https://en.wikipedia.org/wiki/Liskov_substitution_principle

https://en.wikipedia.org/wiki/Covariance_and_contravariance_(computer_science)#Interfaces


What are the implications of returning an interface instead of the concrete type?

- The instance returned will only be able to use the methods defined in the interface, and not the methods defined in the concrete type.
- Allows the developer of the interface to change the type of the object being returned. This means more flexibility for the developer of the implementation.
- What are the advantages for the client?
- If the client needs to cast the interface to concrete types, you might need to examine what is the most frequent use and return the concrete type.


What are the implications of returning a concrete type?
- The clients of the method know exactly the type of the object they're getting. Knowledge of the type allows the clients to know how to best use the type. If for example, the type is a LinkedList you know that random access is slower than in an ArrayList.

https://duncanleung.com/go-idiom-accept-interfaces-return-types/


```java
public ArrayList<String> foo() {
  return new ArrayList<String>();
}
```

Example Usage 1: we want to get the 100th element.
- interface returned: must iterate till th 100th element and return it. Worst we should convert it to List!
- concrete type returned: list.get(100)


Example Usage 2: we want to remove the 100th element.
- concrete type returned: list.remove(100)


## Implications on performance

Returning the concrete type is a winner in this case.

