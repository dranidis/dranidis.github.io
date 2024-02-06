+++
title = 'Tdd Example with Prime Factors'
date = 2024-01-30T23:39:05+02:00
draft = true
+++

The most important benefit of TDD
- You know that your tests fail when something is missing. You saw them failing at least once when you first wrote them.  If you are writing the tests after the completion of the code, assuming your code is correct, you will see the tests passing, *BUT* you will never see them failing. So you can never be sure whether the tests are capturing the failures they are supposed to capture. 

Important rules:
- It is important that each time you write a new test you will see it fail. 
- Write the minimum code that makes the test pass.
- You are allowed to use "copy-paste" in your code to make the test past. After the test passes, examine your code and eliminate the repetition by generalizing.
- Do not generalize before you see the repeating pattern.
- Be careful that you don't implement more than what you are already testing in the existing tests cases. If your implementation contains logic that it is not tested in the current test cases, defer it for the time when you have these more complex test cases.

## Next test case is 2 => [2]

Solve it in the simplest way (don't generalize yet):
```c#
public static List<int> GetPrimeFactors(int number)
{
    List<int> primeFactors = new();
    if (number == 2)
    {
        primeFactors.Add(2);
    }
    return primeFactors;
}
```

## Next test case is 3 => [3]

Repeat what you have done for 2. It's OK to repeat code. You will refactor it at the next step. Don't make big steps.
```c#
public static List<int> GetPrimeFactors(int number)
{
    List<int> primeFactors = new();
    if (number == 2)
    {
        primeFactors.Add(2);
    }
    if (number == 3)
    {
        primeFactors.Add(3);
    }
    return primeFactors;
}
```

## Refactor
Now that you see the repetition pattern, it is time to refactor and generalize. You notice that when the number is 2 or 3, that is if the number is greater than 1 you add the number to the list:
```c#
public static List<int> GetPrimeFactors(int number)
{
    List<int> primeFactors = new();
    if (number > 1)
    {
        primeFactors.Add(number);
    }
    return primeFactors;
}
```
Does this hold for all numbers greater than 1? You don't know yet. You will find out. But for now it feels like the right thing to do.

## Next test case is 4 => [2, 2]
You know that 4 is divided by the prime number 2 (the remainder of the division by 2 is zero), so you have to add 2 in the list and then take the quotient of the division and deal with it. At first you might add the quotient to the list as well.
```c#
public static List<int> GetPrimeFactors(int number)
{
    List<int> primeFactors = new();
    if (number > 1)
    {
        if (number % 2 == 0)
        {
            primeFactors.Add(2);
            number /= 2;
        }
        primeFactors.Add(number);
    }
    return primeFactors;
}
```
But this breaks the test case 2, producing a list [2, 1] where 1 is not a prime number. Good thing that we had the tests to let us know if we are breaking previous cases. So you add another check:
```c#
public static List<int> GetPrimeFactors(int number)
{
    List<int> primeFactors = new();
    if (number > 1)
    {
        if (number % 2 == 0)
        {
            primeFactors.Add(2);
            number /= 2;
        }
        if (number > 1)
        {
            primeFactors.Add(number);
        }
    }
    return primeFactors;
}
```

If you are already thinking that you could use a `while` loop instead of the `if` you are right in terms of the solution, but do you have a test case for the repetition? No, so keep it stupid simple till
you have a test case for 8 => [2, 2, 2], that tests the repetition and then refactor. Additionally, remember that it is OK to repeat code and generalize later, when you clearly see the pattern.

## What is our next test case?

Why don't we consider 5? We know that 5 is a prime number as well, but do we need to 
write a test case for 5? No, because we already have a test case for numbers that are not divisible by 2 and their
prime factors are single element lists. That's the test case for 3 => [3]. 5=> [5] is a similar test case.

What about 6? We know that 6 is divisible by 2, so we have to add 2 in the list and then deal with the quotient. 
We already have a test case for 4 => [2, 2] and 6 => [2, 3] is a similar test case.

What about 7? Same case with 5.

However, 8 is a new case. It is the first test case where the list consists of more than two elements. And the test fails.


## Next test case is 8 => [2, 2, 2]
To make the test pass, you could write:
```c#
public static List<int> GetPrimeFactors(int number)
{
    List<int> primeFactors = new();
    while (number > 1)
    {
        if (number % 2 == 0)
        {
            primeFactors.Add(2);
            number /= 2;
        }
        if (number % 2 == 0)
        {
            primeFactors.Add(2);
            number /= 2;
        }

        if (number > 1)
        {
            primeFactors.Add(number);
        }
    }
    return primeFactors;
}
``` 

## Refactor
We notice the repetition and we generalize it using the pattern. In this case we replace the `if` condition with a `while` loop:
```c#
public static List<int> GetPrimeFactors(int number)
{
    List<int> primeFactors = new();
    if (number > 1)
    {
        while (number % 2 == 0)
        {
            primeFactors.Add(2);
            number /= 2;
        }

        if (number > 1)
        {
            primeFactors.Add(number);
        }
    }
    return primeFactors;
}
``` 

Next test case is 9 => [3, 3]
So what is the smallest change that we can make to make the test pass? We can repeat the same logic we had for 2 and add it to the code:
```c#
    public static List<int> GetPrimeFactors(int number)
    {
        List<int> primeFactors = new();
        if (number > 1)
        {
            while (number % 2 == 0)
            {
                primeFactors.Add(2);
                number /= 2;
            }

            while (number % 3 == 0)
            {
                primeFactors.Add(3);
                number /= 3;
            }

            if (number > 1)
            {
                primeFactors.Add(number);
            }
        }
        return primeFactors;
    }
```
I know, repetition is not good but let's not worry about this yet. Let's
check that the test passes first and we will refactor later.

## Refactor

It feels really nice and safe to have all these test cases. 
We have the same code for 2 and 3. Why don't we generalize using the pattern by adding an external loop?

```c#
public static List<int> GetPrimeFactors(int number)
{
    List<int> primeFactors = new();
    if (number > 1)
    {
        for (int i = 2; i <= number; i++)
        {
            while (number % i == 0)
            {
                primeFactors.Add(i);
                number /= i;
            }
        }

        if (number > 1)
        {
            primeFactors.Add(number);
        }
    }
    return primeFactors;
}
```
## Problem solved?
It seems that are solution works for any case now. We could try a general test case such as: 2 * 2 * 3 * 5 * 11 * 11 => [2, 2, 3, 5, 11, 11]. All numbers are prime numbers.

Are we done? Not yet. Let's examine our solutions and see if we can improve it. No worries, nothing will break. This is why we have all these tests!

## Refactor

The solution seems a bit complex with all these `if` statements that we introduced in the early stages of the solution.

Can we get rid any of the `if` statements?
Let's delete the first `if` statement. The tests still pass.
```c#
public static List<int> GetPrimeFactors(int number)
{
    List<int> primeFactors = new();

    for (int i = 2; i <= number / i; i++)
    {
        while (number % i == 0)
        {
            primeFactors.Add(i);
            number /= i;
        }
    }

    if (number > 1)
    {
        primeFactors.Add(number);
    }

    return primeFactors;
}
```

What if we remove the other `if`? Tests still pass:
```c#
public static List<int> GetPrimeFactors(int number)
{
    List<int> primeFactors = new();

    for (int i = 2; i <= number / i; i++)
    {
        while (number % i == 0)
        {
            primeFactors.Add(i);
            number /= i;
        }
    }

    return primeFactors;
}
```

Now it looks cleaner and less complex. There might be more improvements to do
