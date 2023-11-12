+++
title = 'Database is not memory - ORM - JPA'
date = 2009-10-24T12:42:16+02:00
draft = false
+++

> Note
>
> This is a really old article I have written in blogger.com.
> It appears as it has been posted. Proabably many things are different not at the time of editing (Nov 2024)

This post discusses experiences gathered and realizations made while trying to implement **Object-Relational Mapping** (ORM) for a Java application using [EclipseLink JPA](http://wiki.eclipse.org/EclipseLink) (Java Persistence API).

The application is a Java implementation of the Payroll case study found in the [**"Agile Software Development"** book by R.C. Martin](http://www.objectmentor.com/omTeam/martin_r.html).

_The_ _**payroll** case study_

It's a masterful Object-oriented solution to a realistically complicated payroll system, in which there are Employees who are paid according to different policies. There are salaried, hourly-paid, and commissioned-paid employees. Employees may belong to Union in which they have to pay their dues and service charges.

What is even more masterful about the case study is how Martin presents a **Test-Driven** designed solution.  The design and the coding of the system proceeds incrementally while writing new test cases first and then figuring out how to make them pass.

I need to say that I have first completed all the Java code with all the testing code with no persistence; I used Java Hash tables to store my data in the memory.  The case study is almost completely implemented; things missing are the change transactions (was too lazy to implement them and to be honest, I found them a bit boring).

After completing the code I decided that I need to add some persistence to the case study to make it even more realistic. After some consideration I decided to go with JPA.  By the way this was the first time that I implemented something of that size and complexity with JPA, so probably I have done several mistakes (some of them are documented here for you and me to avoid in the future).  Changing the application to use a database was so easy due to the well thought design.  There is a single Payrolldatabase interface that needs to be subclassed, a Factory class to instantiate the concrete subclass and voila!

And now the story begins....

**"I am so thankful I had all these tests in place"**  
Having all these tests ready to execute at the press of a button made me feel so secure.  I knew that if I was about to do something wrong I would realize it immediately.  I didn't need to check continuously the database to see and examine the entries.

**"JPA is terrific but still..."**  
My initial objective was to leave the Java code as it was and only add the JPA annotations that would do the dirty work.  I didn't want to change ANY of the existing Java code.  I didn't want to mess around with SQL code.  Well, things did not work that way.  It's not a perfect world after all!

- First of all, I needed to add all these default constructors to all my Entity classes.
- Then, I needed to add "artificial" id attributes to Entity classes only for the purpose of a primary key generation. (I initially tried to model referenced instances as Embeddables till I realized that inheritance is not supported; _JPA 2.0 is supposed to support that._)

- I had to turn my interfaces into abstract classes.  Interfaces cannot be entities. And... the whole application is full with interfaces, as it is supposed to be.  _JPA 2.0 is supposed to solve this issue... I will try it._

**"Cascade works only from the parent table to its children" OR "Cascade is not reference"**  
Well, that's perfectly normal.  No one would expect it to work the other way round.  Or would I?  It took me ages to realize that this was the cause to a problem I could not solve.  I had Employee entities cascading changes to their PaymentClassification, Affiliation, and PaymentMethods, ...  mapped entities. So whenever I saved an Employee entity I was sure that everything else was updated in the database.  This worked great when I created Employee objects using the related transactions.  But when I wanted to do other stuff such as adding a TimeCard to an Hourly paid employee the corresponding test cases failed. No TimeCard enitites were saved although the code for that was in place and was working perfectly.  Initially I thought the error was in the way I have annotated the related entities with the OneToMany and Inheritance mappings or the Map I have used in order to store the TimeCard enitites.  I change the annotations millions of times trying every possible combination that could work.  I even started implementing the case from the beginning concentrating on the specific problem and ... it worked in isolation.... TimeCard entries were saved when I was saving the parent entities.  That was the point I realized my mistake. I had to use JPA merge to save the updated employee objects.

This introduced another change in my Java application.  I had to add code to the execute method of ALL the transactions so that the changed employee entities are saved back in the database.  This was clearly not needed when I was working with memory objects since everything is nicely referenced.  You change the insides of an object and you don't need to tell the object that it has changed. Does this make sense?

**"Databases don't have garbage collection" OR "The orphan-removal problem"**  
After running my tests I have found several entries in the database which weren't supposed to be there.  In the case study Employees may have an affiliation or not.  The code uses the Null Object pattern to implement the case of no affiliation.  So an abstract class is defined Affiliation and its two subclasses are the UnionAffiliation and NoAffiliation.  Employee objects are initialized with a NoAffiliation instance, and when saved in the database a corresponding entry is created.  When however employees are associated to a UnionAffiliation instance and the employee object is updated the old NoAffiliation object remains orphan.  In Java it will be eventually be garbage collected. In the database someone (who? probably me!) needs to delete it explicitly.  _I am working on the code to do that....or to find a better solution.  Hibernate supports a_ delete-orphan _annotation but this is not JPA standard. _

- **UPDATE**: JPA 2.0 supports orphan removal. The following annotation solves the problem:  
   @OneToOne(cascade = CascadeType.ALL,orphanRemoval=true)  
   Still, it does not work with Maps.

**"Avoid using Maps"**  
The Java implementation used mainly Maps for the collections.  This was a convenient solution especially for retrieving entries using get. I tried to keep Maps in the JPA annotation. Had really hard time till I made them work and the whole time I was thinking of discarding them and use something else.  The final stroke came with the orphan-removal which did not work with maps, but worked perfectly with Set structures.

**"Forget about debugging"**  
I made so many mistakes on the way and I tried to debug the application to find the causes of my errors.  Why aren't entries saved in the database?  Debugging the application didn't help at all.  There was no way to see what is going on.

The post is already long enough... so probably will not add anything else for the time being.
