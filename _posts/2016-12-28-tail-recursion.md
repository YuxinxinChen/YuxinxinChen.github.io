---
layout: post
title: Tail Recursion
data: 2016-12-28
tags: [Algorithm, Optimization]
comments: true
share: false
---

## Tail Recursion VS Normal Recursion

Use python code just for simplicity.

#### Normal recursion

```python
public int factorial (int x)
{
  if (x == 1)
  {
    return 1;  //base case
  }

  else  /*recursive case*/
   return factorial(x-1) * x;
}
```
What exactly happended when the value of 4 is passed into the function above is the sequence of call made - with the first to last going from top to bottom:

```
factorial (4) ;
return factorial(3) * 4;
return factorial(2) * 3;
return factorial(1) * 2;
return 1;
```
Then, a 2 will be returned (factorial(1) * 2 is equal to 2), and then a 6 will be returned by factorial(2) *3 and so on and so forth until the final value “24” is returned by the function.


#### Tail recursion

```python
def factorial(i, current_factorial=1):
  if i == 1:
    return current_factorial
  else:
    return factorial(i - 1, current_factorial * i)
```

Then:

```
factorial(4, 1)
return factorial (3,  1 * 4 ) //i == 4
return factorial (2,  4 *3 ) //i == 3
return factorial (1,  12 *2 ) //i == 2
return 24  //i == 1
```

## The difference between tail recursion and normal recursion
You can see that once we make a call to factorial (1, 12 * 2 ), it will return the value of 24 – which is actually the answer that we want. The most important thing to notice about that is the fact that all of the recursive calls to factorial (like factorial (2, 4 * 3 ), factorial (3, 1 * 4 ), etc) do not actually need to return in order to get the final value of 24 – you can see that we actually arrive at the value of 24 before any of the recursive calls actually return. So, we can say that a function is tail recursive if the final result of the recursive call – in this case 24 – is also the final result of the function itself, and that is why it’s called “tail” recursion – because the final function call (the tail-end call) actually holds the final result. If we compare that with our earlier example of “normal” recursion, you should see the difference – the “normal” recursive call is certainly not in it’s final state in the last function call, because all of the recursive calls leading up to the last function call must also return in order to actually come up with the final answer.

## Tail recursion optimization
In our tail recursive example, the recursive calls to factorial do not actually **need** a new stack frame for each and every recursive call that is made. This is because the calculation is made within the function parameters/arguments – and the final function call actually contains the final result, and the final result does not rely on the return value of each and every recursive call. However, an important point that you should make sure you understand is that **it is up to the compiler/interpreter of the particular language to determine whether or not the recursive calls in a tail recursive function actually use an extra stack frame for each recursive call to the function.**

Some language’s compilers choose to optimize tail recursive functions because they know that the final result will be in the very last function call. So, the compilers will not create a new stack frame for each recursive call, and will instead just re-use the same stack frame. This saves memory overhead by using a constant amount of stack space, which is why it is called tail recursion optimization. But, keep in mind that some compilers do not perform this optimization on tail recursive functions, which means that the tail recursive function will be run like a normal recursive function and will use a new stack frame for each and every function call.

##### Python does not use tail recursive optimization
In our example of tail recursion, we used Python because it made it easy to illustrate our example. But, it’s interesting to note that Python does not actually utilize tail recursive optimization, which means that it treats tail recursive calls just as it would treat normal recursive calls. This of course means that tail recursive calls in Python will be less efficient then they would be if they were optimized, but there are valid reasons the creator of Python decided not to add this feature.

##### However Scala enables tail recursion optimization
To use tail recursion optimization, you need tell compiler by:

```scala
import scala.annotation.tailrec

def factorial(n: Int): Int = {
@tailrec def factorialAcc(acc: Int, n: Int): Int = {
if (n <= 1) acc
else factorialAcc(n * acc, n - 1) }
      factorialAcc(1, n)
}
```
Note that you can use the @tailrec annotation in situations like this to confirm that your algorithm is tail recursive. If you use this annotation and your algorithm isn’t tail recursive, the compiler will complain. More information:<a href="http://blog.richdougherty.com/2009/04/tail-calls-tailrec-and-trampolines.html"> tail recursion and trampolines </a>, <a href="http://stackoverflow.com/questions/1187433/scala-factorial-on-large-numbers-sometimes-crashes-and-sometimes-doesnt"> tail-call optimization in Scala </a>


##### Tail Recursion versus Iteration
Tail recursion can be as efficient as iteration if the compiler uses what is known as tail recursion optimization. If that optimization is not used, then tail recursion is just as efficient as normal recursion.

##### Tail recursion optimization and stack overflow
Because tail recursion optimization essentially makes your tail recursive call equivalent to an iterative function, there is no risk of having the stack overflow in an optimized tail recursive function.

##### Tail call optimization versus tail call elimination
Both tail call optimization and tail call elimination mean exactly the same thing and refer to the same exact process in which the same stack frame is reused by the compiler, and unnecessary memory on the stack is not allocated.


