---
layout: post
title: Countdown
---

Do do-do do do-do, do-do do-do do-do da-do.

<center><hr style="width:50%"></center>
<br>

Countdown is a **very** long-running UK quiz show[^1], based on word and number puzzles. There are three parts to parts to each game: i) ten letter rounds, where the contestants try to make the longest possible words from nine random letters; ii) four number rounds, where the contestants try to make a target number from six other numbers using simple arithmetic, iii) the contestants solve a nine-letter anagram.

During lockdown I ended up watching a fair bit of Countdown, and given that I had a lot of time on my hands I decided to write some code to play the game.


# Letters round

For the letters round we get nine random letters, where we get the choice of how many are vowels and how many are consonants. Also, letters aren't chosen uniformly at random, they are stratified with respect to the frequency of the letters use in English. Let's say we've got:

$$
\rm{A\:R\:I\:O\:E\:K\:L\:N\:N}
$$

How can we find the longest word from this set of letters? The way that I've done it is by computing the "letter set" for each word - the sorted set of letters in the word - and constructing a dictionary mapping letter sets to the list of words that share that letter set. An example might be helpful here, take the word `cats`. The letter set of `cats` is `acst`, and this letter set can also be used to make three other words, so the entry in our dictionary is

```
'acst' : ['acts', 'cast', 'cats', 'scat']
```

The letter set map is made like this, where `WORDS` is the contents of `/usr/share/dict/british-english`

```python
WORD_MAP = defaultdict(list)

join = "".join
def key(x): return join(sorted(x))

for w in WORDS:
    WORD_MAP[key(w)] += [w]
```

Then to find all of the words for a given board we just need to get all of the letter sets (of different lengths) from our nine letters, look them up in the dictionary and pick the longest words that result.

```python
def lettersets(letters: Letters) -> Set[Word]:
    return {key(l) for i in range(4, 9) for l in combinations(letters, i)}

def best_words(letters: Letters, n: int = 5) -> Words:
    poss = [WORD_MAP[s] for s in lettersets(letters) if s in WORD_MAP]
    return sorted(sum(poss, []), key=len, reverse=True)[:n]
```

Simple as.

For the letters given above we get,

```
best_words("arioeklnn")
['lankier', 'aileron', 'alienor', 'oarlike', 'kaoline']
```

# Numbers round

The numbers round is a tougher problem. Here we get 6 random numbers, where we can chose how many are small ($\le 10$) and how many are large ($(25, 50, 75, 100)$). We're then given a target between $100$ and $1,000$. The goal is to get as close as possible to the target using the four basic operations (addition, subtraction, multiplication, and division). We don't have to use all six numbers, and we can't at any point use a non-natural number (a fraction or negative number).

Let's have an example. 

$$
\rm{Numbers=}\:(1,\, 8,\, 9,\, 6,\, 50,\, 100) \:\: \rm{Target=}\:836
$$

Is it possible to make the target by doing simple arithmetic operations to the numbers given?

There are lots of clever ways of doing this (I'm sure) but I'm going to do it the simple way,by brute-force checking all of the possible equations we can make from those numbers. To do this I'll encode the equations using Reverse Polish Notation[^2], which makes life easier because it means we don't have to worry about placing parentheses to denote the order of operations. In RPN an equation is just a list of $N$ numbers and $N - 1$ operators:

$$
(6, 8, 9, +, \times) \rightarrow ((9 + 8) \times 6)
$$

All we need to do it take the product of all length $N$ permutations of numbers and length $N-1$ permutations of operators, and check if any evaluate to the number we want. Generating the equations isn't too hard:[^3]


```python
def eqn_nums(nums: Numbers) -> Equations:
    ops = combinations_with_replacement("+-/*", len(nums) - 1)
    op_perms = mapcat(permutations, ops)
    num_perms = set(permutations(nums))

    eqns = product(num_perms, op_perms)
    return (list(p) + list(o) for p, o in eqns)


def all_eqn(nums: Numbers) -> Equations:
    for i in range(4, len(nums) + 1):
        for n in combinations(nums, i):
            yield from eqn_nums(n)

```

Now to evaluate the equations we just need an RPN calculator. I won't show that here (see footnote 2), but it's quite a simple piece of code. And one nice thing about is that we can enforce our _natural numbers only_ rule.

Now we can just use a bit of code to rewrite our RPN in infix for display and we're done.[^4] 

```python
def is_solution(nums: Numbers, t: int) -> bool:
    try:
        return calc(nums) == t
    except NotValidEqnError:
        return False


def solution(nums: Numbers, t: int) -> Numbers:
    return next(filter(lambda x: is_solution(x, t), all_eqn(nums)), [''])


def to_infix(expr: Expression) -> str:
    stack = []
    for s in expr:
        if s not in ops:
            stack = [s] + stack
        else:
            subexpr = f"({stack.pop(0)} {s} {stack.pop(0)})"
            stack= [subexpr] + stack

    return stack[0]

to_infix(solution([1, 8, 9, 6, 50, 100], 836))
```

Giving

$$
((((100 - 1) \times 8) - 6) + 50) = 836
$$


# Conundrum

The final part of the game is the Countdown Conundrum, which is a nine-letter anagram. To solve this anagram we can just use the code from our letters round. The real fun is making the conundrum, the nine-letter anagram being made up of a four and five-letter word. To do this we get all the nine-letter words (but only if you can't make any other nine-letter words from those letters, to avoid confusion), we get all of the four and five-letter words, and then return a triple of (four, five, nine-letter) words once we find four and five letter words that make a nine-letter. Probably clearer in code

```python
def conundrums():

    uniq9s = {k: v for k, v in WORD_MAP.items() if len(v) is 1 and len(k) is 9}
    fours = shuffled(w for w in WORDS if len(w) is 4)
    fives = shuffled(w for w in WORDS if len(w) is 5)

    for fr, fv in product(fours, fives):
        k = key(fr + fv)
        if k in uniq9s:
            yield fr, fv, uniq9s[k][0]
```

Running this gives us something like,

$$
(\rm{lets}, \:\: \rm{mania}) \rightarrow \rm{laminates}
$$

So there we go, that's all we need play Countdown. The code is [here](https://github.com/neal-o-r/countdown), a nice way to spend lockdown.

---

[^1]: There have been more the 7,000 episodes (!) and the show it's based on (Des chiffres et des lettres) has been running on French TV continuously since 1965.

[^2]: I've used this to solve a [related problem](https://n-o-r.xyz/2019/02/20/licence-plates.html) before.

[^3]: Two things you'll notice here: i) we don't take equations with fewer than four numbers, and ii) we take the set of the number permutations to avoid doing redundant calculations when we have duplicate numbers.

[^4]: Interestingly one thing I tried was just writing the equations in infix and `eval`-ing them, saving on needing an RPN calculator. It turned out this was ~10x slower.
