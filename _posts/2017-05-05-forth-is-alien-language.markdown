---
layout: post
title:  "Forth is alien language"
date:   2017-05-05 23:50:46 -0500
categories: forth
---
Forth is from like 1970, and is a "stack-based" programming language. Basically, the program has one stack that you can put values on and take values off of, instead of you getting to use variables with names (you can actually use variables with names, but they're discouraged because *all* of them are global). Functions, instead of taking arguments and returning values, operate on the stack and put the results on the stack. Like haskell, I feel like it is alien language.

If I say a number value, it gets put on the stack. If I say a function name, it calls that function. If I want to do 3 + 4, I say

    3 4 + .

3 gets put on the stack, 4 gets put on the stack, '+' gets called which grabs the top two numbers off the stack and adds them and puts the result on the stack, and "." which takes the top number off the stack and prints it. If I wanted to do 3 * 4 + 5, I would say

    3 4 * 5 + .

3 and 4 get added to the stack, taken off and multiplied (now the stack is just "12"), 5 is put onto the stack (now it's "12 5") and "+" adds the 12 and 5 putting 17 back onto the stack.

There are some operations that are provided to manipulate the stack. "dup" takes the thing on the top of the stack, and adds it to the stack again, so if I said "1 2 3 dup" the stack would be "1 2 3 3". "over" takes the second thing on the stack, and copies it onto the top of the stack (after "1 2 3 over" the stack would look like "1 2 3 2"). "swap" swaps the top two items. "drop" takes the top thing off the stack and does nothing with it.

In order to define functions you say

    : <function_name> <op1> <op2> <op3> ;

so I could define a function that squares a number like this:

    : square dup * ;

Now when I say

    5 square .

... it goes like this: I add 5 to the stack. I call the square function, which duplicates the top number (now my stack is "5 5") and calls "*", which grabs my initial 5 and the new one and multiplies them (now the stack is "25"). Now the function is done, and we're back to the line I called it from, where the dot takes that last 25 off the stack and prints it.

What if I wanted to do 3^2 + 4^2? Now that square is defined I could do this in one line like this:

    3 square 4 square + .

but I could also define a function like this

    : sum_of_squares square swap square + ;

... and call it...

    3 4 sum_of_squares .

... and here's how this will go. I added 3 and 4 to the stack and called sum_of_squares, which:

- calls square, which takes the top number 4 off the stack and puts 16 back (now the stack is "3 16")
- swaps the top two items on the stack (now it's "16 3")
- squares the top number on the stack (now it's "16 9")
- adds the top two numbers on the stack (now the stack is just 25)

Finally, I grab 25 off the stack and print it. Since this only operates on the top of the stack, if I had started with "1 2 3" instead of an empty stack, it would all work the same, and at the end I would pop 25 off of the stack "1 2 3 25" and be left right where I started.

sum_of_squares is basically a function that takes two arguments, but I only know that because I know what the function does with the stack. Since when you define or call a function, there's nothing in the syntax to suggest what exactly it's doing to the stack, it's customary to include a comment in the function definition to show this:

    : sum_of_squares ( a b -- a^2 + b^2 ) square swap square + ;

The opening parenthesis is actually a function that says, ignore everything until a closing parenthesis. The space after it is necessary, otherwise interpreter would look for a function called "(a"!

Booleans are number values, where True means all the bits are set to 1, and False means they're all set to 0. They're actually signed integers though, so True ends up being represented as -1. This means that "0 1 =" will put a 0 on the top of the stack, and "1 1 =" will put a -1.

If-statements (which have to be in function definitions) look like this:

    if <what to do if true> then

OR

    if <what to do if true> else <what to do if false> then

and the condition is whatever was on top of the stack when you get to "if" (in practice anything that's not 0 is considered true). Comparisons are functions that use the top two numbers on the stack just like arithmetic, so you could write an absolute value function like this:

    : abs2 ( a -- |a|)
        dup 0 < ( set up the condition: put a boolean on top of the stack indicating whether the top number is negative)
        if
            0 swap - ( if negative, subtract from 0)
        then
    ;

Looping takes the last thing on the stack as a starting value, and the next one down as a limit, and looks like this:

    do <what to do> loop

Like "if", it has to be inside a function definition. While the loop is running, the function capital "I" puts the current value on the stack, so

    : print-nums 10 0 do I . loop ;
    print-nums

prints "0 1 2 3 4 5 6 7 8 9". This means we could do a factorial function like this:

    : factorial
        dup 1 + ( set up the end value)
        1 ( start at 1)
        do
            I * ( multiply the top number on the stack by the count)
        loop
    ;

We could also do it- recursively! When defining a function that calls itself, the forth interpreter gets confused because it notices that as you define the function you're using a symbol that's not defined yet. The alternative is to use a function called "recurse"! Here it is in python:

{% highlight python %}
def factorial(n):
    if n == 0:
        return 1
    else:
        return n * factorial(n - 1)
{% endhighlight %}

and here's roughly the same thing in forth:

    : factorial
        dup 0 > ( are we taking factorial of a number > 1?)
        if
            dup 1 - recurse * ( add the next number to the stack and recurse)
        else
            drop 1 ( replace the 0 with a 1)
        then
    ;

As this descends recursively, it builds a list of numbers on the stack, from wherever all the way down to 0, and after each function call returns we multiply two more of the numbers together.

Now here is a tricky implementation detail. Because python, c, javascript whatever have variables, when you call a funtion the whole scope that you have at that moment has to be saved, along with a "return pointer" which tells the computer where to go to resume the previous function, and you end up with a big ol' stack of these environments and return pointers. With forth, where we have no named variables, the entire scope has been reduced to a single number on the stack, but it does still have to remember where to go to pick up at the previous function, somehow. It turns out there's a second stack, called the "control" or the "return" stack where it stores all this.

As a programmer you can put actual values on the control stack- the function '>R' takes a value off of the "parameter" stack and puts it onto the control stack, and 'R>' does the opposite (although one book I found warns "Such use is discouraged for novices since it adds the spice of danger to programming")

Loops actually use this too- the "I" you can use to get the current value during a loop is actually grabbing that value from the top of the return stack ('J' gets the third value down, which is useful for nested loops).

If we were doing factorial in scheme, we would make it "tail recursive", making sure that there weren't any calculations that happened after our function call, so that the interpreter could forget our scope. In scheme that looks like this:

{% highlight scheme %}
(define (factorial n) (fact-iter n 1))
(define (fact-iter n product)
    (if (= n 0)
        product
        (fact-iter (- n 1) (* product n))))
{% endhighlight %}

fact-iter passes the product-so-far to the next recursive function call in an argument, so that the interpreter doesn't have to save the environment or return pointer- whatever the innermost function returns, is what the outermost function returns. (I also added a wrapper function so that you don't always have to pass in 1 the first time you call it). In python it would look like this:

{% highlight python %}
def fact_iter(n, product):
    if n == 1:
        return product
    else:
        return fact_iter(n - 1, product * n)

def factorial(n):
    return fact_iter(n, 1)
{% endhighlight %}

This might be a little more readable, but since its syntax is more complicated (or maybe because you could just use a loop) python doesn't notice when you're doing something tail recursive (Scheme does actually notice this).

Here's a similar thing in forth:

    : fact-rec ( n product -- fact n)
        over 0 >
        if
            over * swap 1 - swap recurse
        then
    ;
    
    : factorial ( n -- fact n)
        1 ( add our product argument)
        fact-rec
        swap drop ( drop the counter, so only the result is left)
    ;

Now we use the top number on the stack like the product-so-far argument in scheme and python, and the second number down on the stack as a counter. The wild thing is, the interpreter doesn't need to recognize that it's tail recursion- the parameter stack stays 2-deep until the "factorial" function drops the counter and leaves just the final result. The return stack still keeps track off the whole series of recursive calls, so it still takes O(n) memory.

PHEW