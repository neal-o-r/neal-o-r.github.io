---
layout: post

title: Áireamhán
---

Teanga ríomhchlárúcháin as Gaeilge.


About a year ago I decided it would be a good idea to make a programming language. More specifically I thought, since one didn't exist, I should make a programming language in Irish. And since Lisps are really cool, I thought this language should probably be a Lisp. Taking a lot of inspiration from some smart folks[^1], I built [Áireamhán.](https://github.com/neal-o-r/aireamhan)

In this blog post I want to sum up what a Lisp is and give an idea of how they work, and by extension other how Áireamhán works. It's loosely based on a talk I've given a few times, which you can watch [here](https://www.youtube.com/watch?v=0r8eIU_aJsY). The language itself is implemented in Python, so if you want to play around with it you can get it from PyPI: ```pip install aireamhan```.

<center>
<img src="/images/aireamhan/aireamhan.png">
</center>

<br>

<center><hr style="width:50%"></center>
# Lisp

Lisp is a very old language. In fact it's the second-oldest programming language in widespread use, younger only than Fortran. It was developed by John McCarthy, coiner of the phrase AI, at MIT in the late 1950's.

<center>
<img src="/images/aireamhan/jmc.jpg" width="500px">
</center>

<br>
There he is, hard at it.[^2] In McCarthy's time Fortran was the dominant language, the only show in town for most purposes. He was working on problems in symbolic logic -- Good Old Fashioned AI -- and found that Fortran wasn't suitable. Deciding he needed a better tool for his research he began work on Lisp. An early inspiration was the language IPL, and from it he took the idea to store expressions as lists of symbols (hence Lisp, <b>Lis</b>t <b>P</b>rocessing). Finding the ```if``` statement in Fortran very cumbersome, McCarthy invented an alternative conditional expression, which returns sub-expression $A$ if the supplied test succeeds and sub-expression $B$ if the supplied test fails, and which also only evaluates the sub-expression that is returned. He also found, when trying to write a program to perform differentiation, that it would be useful to have a function that would take another function as an argument and apply it to all the elements in a list, the ```map``` function. These 3 simple ideas led to a lot of the power of Lisp: the list syntax leads to the S-expression notation for data and function application, building on the conditional operator we get recursion, and, with ```map``` as a starting point, higher-order functions.[^3] These 3 innovations alone were enough to make Lisp radically different from anything that had gone before, and have inspired many languages since.

In 1958 McCarthy set some students to work at implementing Lisp and in 1960 he published a classic paper, *[Recursive Functions of Symbolic Expressions and Their Computation by Machine](http://www-formal.stanford.edu/jmc/recursive.pdf)*, which presents the idea behind Lisp both as a programming language and as a way "of describing computable functions much neater than [the] Turing machines". In this paper he boils down the core of the language to a set of axioms, and it's worth taking the time to walk through them and seeing how they come together.

He begins by defining 2 primitives; firstly a Boolean ```true```, and secondly that the empty list evaluates to nothing,[^4]  ```() => nil```. Building out from this core he defines 7 simple functions:

#### 1 - <tt>quote</tt>:

This one can seem a little unusual to people new to Lisp, but the first function we define simply returns its argument unaltered

<center>
<code>(quote ()) => ()</code>
</center>
<br>

So rather than evaluating the empty list to ```nil``` we just get the empty list back. This is useful because it gives us a way to introduce data into the language off the bat, the argument to ```quote``` need not be a valid list expression, since it won't be evaluated it can be whatever we want, for instance ```(quote (1 2 3)) => (1 2 3)```.

#### 2 - <tt>atom?</tt>:

Next we introduce a Boolean operator that tells us if an input is *atomic*, by which we mean is it a symbol or an empty list, something that can't be broken up

<center>
<code>(atom? ()) => true<br>
(atom? (quote (1 2 3))) => nil</code>
</center>
<br>

#### 3 - <tt>eq?</tt>:

We define a equality operator to check if two inputs evaluate to the same thing,

<center>
<code>(eq? () ()) => true</code>
</center>
<br>

Notice here the prefix notation that Lisp uses. Defining things with this notation greatly simplifies the parsing of expressions, as well see later.

#### 3 - <tt>car</tt>:

The ```car``` function (short for content addressable register) will be familiar to people who've worked with functional languages before, it just gives the first element of a list

<center>
<code>(car (quote (1 2 3)) => 1</code>
</center>
<br>

#### 4 - <tt>cdr</tt>:

The opposite of ```car```
<center>
<code>(cdr (quote (1 2 3)) => (2 3)</code>
</center>
<br>


#### 5 - <tt>cons</tt>:

<code>cons</code> is another familiar function, it's used to make lists

<center>
<code>(cons 1 ()) => (1)</code>
</center>
<br>

#### 6 - <tt>if</tt>:

The ```if``` statement we've previously discussed, with its succinct definition

<center>
<code>(if cond A B)</code>
</center>
<br>

Returns $A$ if the condition is true, otherwise it returns $B$.

#### 7 - <tt>lambda</tt>:

Finally we define a ```lambda```, which is common to lots of languages, this is how we define functions in Lisp

<center>
<code>(lambda (args) expr)</code>
</center>
<br>


Using these 7 simple pieces McCarthy does something very cool; he defines 2 functions ```apply``` and ```eval```, you can see them on page 13 of the Lisp Manual below, which can parse and evaluate any arbitrary Lisp expression.

<center>
<img src="/images/aireamhan/eval.png" width="500px">
</center>


Isn't that amazing? Those 7 functions, combined the right way, are all you need to define a Lisp interpreter. So to bring it all back to Áireamhán, all I needed to do was implement those function in Irish. Easy!

<br>
<center><hr style="width:50%"></center>
<br>

# Python

We need a few pieces to build a Lisp, here's how we go about doing it in Python.[^5]

The first thing we is a parser, something that will turn strings of characters into lists of tokens. Thanks to the really simple syntax (all those wonderful parentheses!) we can do this with a regular expression

```python
line = '(car (quote (1 2 3)))'

import re
tokenizer = r"""\s*([(')]|"(?:[\\].|[^\\"])*"|;.*|[^\s('";)]*)(.*)"""

for i in range(11):
    char, line = re.match(tokenizer, line).groups()
    print('{}, "{}"'.format(char, line))
```

This regex will allow us to break that input string up token by token.

With this simple regex we can tokenize an input, and once we've go the tokens we can parse them into a nested list structure called a parse tree, a hierarchical structure we can evaluate from bottom to top,

```python
def parse(input_line):
    def read_ahead(token):
        if '(' == token:
            L = []
            while True:
                token = input_line.next_token()
                if token == ')':
                    return L
                else:
                    L.append(read_ahead(token))
        elif token == ')':
            raise SyntaxError('BAD SYNTAX')
        else:
            return token

    token1 = input_line.next_token()
    return read_ahead(token1)

tree = parse(tokenize('(car (quote (1 2 3)))'))
print(tree)
```
```
>> ['car', ['quote', ['1', '2', '3']]]
```  

Now that we have the parse tree we're pretty much home. Now it's just a matter of implementing an ```eval``` function, using the 7 primitive functions we found above, and we've got a Lisp. Here's an example, using just two functions:

```python
def evaluate(tree):

    if tree[0] == 'car':
        return evaluate(tree[1])[0]

    if tree[0] == 'quote':
        return tree[1]

evaluate(tree)
```
```
>> '1'
```

Done! Of course to get a full Lisp you'll need the rest of those functions, and a slightly more complex ```eval``` function, but that's the basic idea in a nutshell.

<br>
<center><hr style="width:50%"></center>
<br>

# Áireamhán

It goes without saying that with all that done the sensible thing to do is translate those function names into Irish, and give the world it's first and only programming language *as Gaeilge*. So to finish off here's an example of what Áireamhán looks like. The example I'll give is a simple recursive factorial function, and I'll walk through the syntax to explain what's going on at each step. Here's the function:

```
(sainigh factorial (lambda (n)
  (má (= n 1)
    1
    (* n (factorial (- n 1)))
)))
```

The first line is the function signature, we use the keyword ```sainigh``` (define) to bind functions or values -- Áireamhán doesn't distinguish between the two -- to variables. In this case we bind a lambda to the name ```factorial```. This lambda takes one input, $n$. The body of the code is just one of those very tidy ```if``` statements (```má``` being Irish for if). If $n = 1$ (note the prefix notation) then we return $1$, otherwise we recursively compute the factorial of $n-1$ and take it's product with $n$. Very simple. And very syntactically similar to Python itself, take out the parentheses and convert to infix and you're pretty much looking at Python code.

<br>
<center><hr style="width:50%"></center>
<br>

# Focal scoir

Lisp is a pretty unique language. In some sense it's more than just a language, it's a way to think about specifying and carrying out computation. It breaks those ideas down into their axioms. And it shows, somewhat surprisingly, that you only need a few very simple pieces to make a symbolic system that's able to understand and execute its own symbols. Thinking about programming from this perspective is very powerful. Things that sounds complicated, like building a programming language, become much clearer when viewed through the lens of McCarthy's big idea.


---

[^1]: Mostly, [Peter Norvig](http://norvig.com/lispy2.html), [Paul Graham](http://www.paulgraham.com/lisp.html), and [Michael Nielsen](http://www.michaelnielsen.org/ddi/lisp-as-the-maxwells-equations-of-software/)

[^2]: With a name like McCarthy you might guess that he has some Irish ancestry, and I discovered when writing Áireamhán that his father was actually born and raised in Cromane, near to the West Kerry Gaeltacht.

[^3]: All of the Lisp development history is well-documented, McCarthy himself gives a good [summary](http://jmc.stanford.edu/articles/lisp/lisp.pdf).

[^4]: The ```nil``` symbol double-jobs as Boolean False.

[^5]: The example code used here can all be found in this [notebook](https://github.com/neal-o-r/aireamhan_slides/blob/master/%C3%A1ireamh%C3%A1n.ipynb).
