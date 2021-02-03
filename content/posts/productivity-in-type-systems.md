---
title: "Productivity in Type Systems"
date: 2021-01-30T19:00:00-05:00
draft: false
---

One of the commonly cited drawbacks of type systems is that they can inhibit 
productivity. I think this depends on your definition of productivity. I can 
think of two:

1. To write the smallest amount of code in the shortest amount of time
2. To consistently produce working and bug-free code in such a way that it 
scales as the codebase and team grow over time.

The presence of a type system could certainly hurt productivity in the first
case, because languages with type systems often need to be supported by 
compilers and build tools. I find this type of productivity to be useful when
I'm writing small scripts that are run only a handful of times and/or rarely
change.

Type systems are invaluable in the second case because they offer benefits
that support those dimensions of scaling:
1. They provide a framework within which you can precisely express your ideas.
My favorite example is _algebraic data types_ or _enums_, which are a 
fundamental construct in data modeling, but are absent in most mainstream
languages.
2. They guarantee the absence of certain errors in a well-typed program, 
particularly large programs which are rapidly changing.
3. Typing information serves as documentation and as a static approximation 
for how a program behaves, so it is easier to infer intent from code long 
after it is written.
