+++
title = 'Returning and Accepting Types'
date = 2024-02-03T02:15:20+02:00
draft = true
+++

https://enterprisecraftsmanship.com/posts/return-the-most-specific-type/

https://en.wikipedia.org/wiki/Liskov_substitution_principle

https://en.wikipedia.org/wiki/Covariance_and_contravariance_(computer_science)#Interfaces



# [[Robustness principle - Postel's law]]

In [computing](https://en.wikipedia.org/wiki/Computing "Computing"), the **robustness principle** is a design guideline for software that states: 

> [!Quote] 
> "be conservative in what you do, be liberal in what you accept from others".  #quote

 It is often reworded as: 
 

> [!Quote]
>  "be conservative in what you send, be liberal in what you accept".  #quote
 
 The principle is also known as **Postel's law**, after [Jon Postel](https://en.wikipedia.org/wiki/Jon_Postel "Jon Postel"), who used the wording in an early specification of [TCP](https://en.wikipedia.org/wiki/Transmission_Control_Protocol "Transmission Control Protocol").[[1]](https://en.wikipedia.org/wiki/Robustness_principle#cite_note-1)


 ## Prefer interfaces over concrete types

 This definitely holds for our parameters when we design a function.
 
 Let's assume a function that accepts an argument and performs some calculation. In a dynamically typed language the definition might be:
 ```
 calculate(arg) {
    ...
    int x = a.mtd();
    ...
 }
 ```

 Any type that offers the method `mtd` could call the function. The implicit interface of the arg is
 ```
 interface A {
    int mtd()
 }
 ```

 You want your clients to be able to use the function `calculate` for any argument that satisfies the interface. 
 There is no reason, and it is a bad practice to restrict your clients.

 In a typed language the best approach would be to explicitly define the interface A and then define the function as:
 ```
 void calculate(A arg) {
   ...
 }
 ```
 
 What about return types?
