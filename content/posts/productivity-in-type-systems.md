---
title: "Productivity in Type Systems"
date: 2021-01-30T20:00:00-05:00
draft: false
---

One of the commonly cited drawbacks of type systems is that they can inhibit 
productivity. I think this depends on your definition of productivity. I can 
think of two:

1. To write the smallest amount of code in the shortest amount of time.
2. To consistently produce working and bug-free code in such a way that it 
scales as the codebase and team grow over time.

The presence of a type system could certainly hurt productivity in the first
case, because languages with type systems often need to be supported by 
compilers and build tools. 

Type systems are invaluable in the second case because they offer benefits
that support those dimensions of scaling:
1. They provide a framework within which you can precisely express your ideas. 
You can easily infer intent from code, which I find is hard to do in languages 
like vanilla JavaScript and Python.
2. They guarantee the absence of certain errors in a well-typed program.
3. Type names and annotations serve as documentation and as a static 
approximation for how a program behaves.
