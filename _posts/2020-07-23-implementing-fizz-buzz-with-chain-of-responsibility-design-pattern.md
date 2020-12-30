---
layout: post
title:  "Implementing fizz buzz with the chain of responsibility design pattern"
date:   2020-07-09 13:37:00 +0100
---
Fizz buzz is a game played by children to learn about division. The idea is that players take turns incrementing a number, 
then replacing the numbers divisible by 3 and 5 with 'fizz' and 'buzz' respectively. 

This problem is fairly easy to program, so I am not going to show the basic implementation in this post.

What I want to do is using the 'Chain of Responsibility' design pattern to tackle this problem. The chain of responsibility 
design pattern is a behavioral design pattern in which a chain of objects will try to handle a request. This way you can 
easily add new handlers without worrying about breaking the old code.

## The problem
I have the following problems to solve when implementing the fizz buzz challenge: 

- If a number is divisible by 3 and 5 show 'fizzbuzz' instead of the number.
- If a number is divisible by 5 show 'buzz' instead of the number.
- If a number is divisible by 3 show 'fizz' instead of the number.
- If a number does not meet any of the above criteria show the number.

## Solving the problem using the chain of responsibility
The first thing I created was an interface `FizzBuzzSolverInterface`
```php
interface FizzBuzzSolverInterface 
{
    public function solve(int $number): string;
}
```

Then I implemented the other classes of the chain, this is the object which will be used in the rest of the code for when
I have to solve a number. You can choose to omit this class and call the next first class, I just think this is a nice 
entry point.

```php
class FizzBuzz implements FizzBuzzSolverInterface 
{ 
    private FizzBuzzSolverInterface $next;
    
    public function __construct(FizzBuzzSolverInterface $next) 
    {
        $this->next = $next;
    }

    public function solve(int $number): string 
    {
        return $this->next->solve($number);
    }
}
```
The rest of the classes are almost the same as this class, except for their implementation of the `solve()` method. So 
the other classes look like this. 

```php
class DivisibleByThreeAndFive implements FizzBuzzSolverInterface 
{
    public function solve(int $number): string 
    {
        if ($number % 3 === 0 && $number % 5 === 0) {
            return "fizzbuzz";
        }   
        return $this->next->solve($number);
    }
}

class DivisibleByThree  implements FizzBuzzSolverInterface
{
    public function solve(int $number): string 
    {
        if ($number % 3 === 0) {
            return "fizz";
        } 
        return $this->next->solve($number);
    }
}
class DivisibleByFive implements FizzBuzzSolverInterface 
{
    public function solve(int $number): string 
    {
        if ($number % 5 === 0) {
            return "buzz";
        } 
        return $this->next->solve($number);
    }
}

class NoDivision implements FizzBuzzSolverInterface
{
    public function solve(int $number): string 
    {
        return $number;
    }
}
``` 

Now we have all the pieces of the chain we can assemble them, and iterate over a list of numbers to play the fizz buzz game.
We instantiate the classes in the right order and call the `solve()` method on the `FizzBuzz` class.

```php
$fizzbuzz = new FizzBuzz(
    new DivisibleByThreeAndFive(
        new DivisibleByThree(
            new DivisibleByFive(
                new NoDivision()
            )
        )
    )
);

// will echo '12fizz4buzzfizz78fizzbuzz11fizz1314fizzbuzz'
foreach(range(1, 15) as $number) {
    echo $fizzbuzz->solve($number); 
}

```  

Make sure the handlers which have priority over other handlers are higher up in the chain. In my case the  `DivisableByThreeAndFive` 
handler has to go before the `DivisableByThree` and `DivisableByFive` handlers because otherwise, the word 'fizzbuzz' will never be returned.

While this works, this is not very flexible. I used the [Symfony service container](https://symfony.com/doc/current/service_container.html)
to wire all the classes. So I added this to my `service.yaml`.

```yaml
FizzBuzz:
    arguments:
        $next: '@DivisibleByThreeAndFive'

FizzBuzzMultipleOfThreeAndFive:
    arguments:
        $next: '@DivisibleByThree'

FizzBuzzMultipleOfThree:
    arguments:
        $next: '@DivisibleByFive'

FizzBuzzMultipleOfFive:
    arguments:
        $next: '@NoDivision'
```

If you want to also divide by 7 then it's a matter of creating the handler, add 3 lines to the `service.yaml` and you
are done.

## Conclusion
The chain of responsibility design pattern is great when you have to solve a problem in which you have to do multiple 
checks on the same object in a row. 

Sometimes you will see implementations of the chain of responsibility design pattern
where not all requests will be handled, this can be either intentional or accidental. When it's accidental it mostly because
a handler forgets to call the `$next` handler. 

The example is a bit over-engineered, but it gives a good example of how the pattern works was a fun exercise 
with this design pattern.
