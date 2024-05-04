---
layout: post
title:  "Always lint python typing"
date:   2024-05-02 20:19:37 +0100
categories: python, testing
---

## Dynamically and statically typed languages

Unlike statically typed languages like Java, C++ and others, Python is a dynamically typed language. This means that the type of the variable isn't fixed. To illustrate this, let's look at some valid Python code in the following example.

```
def addOne(number) :
  return number + 1

print(addOne(5)) # prints '6'
print(addOne("5")) # TypeError: can only concatenate str (not "int") to str
```

So while it does throw an error when the code is executed, it won't prevent you from packaging and deploying your code. However, the same thing is not possible in a statically typed language like java:

```
int addOne(int num) {
    return num + 1;
}

void main() {
  System.out.println(addOne(1));
  System.out.println(addOne("1"));
}
// error: error: incompatible types: String cannot be converted to int
```

This is a compile error, so even if we'd want to, we cannot run this code. The compiler doesn't let us. In many cases, this is an advantage. It's impossible to accidentally pass a number to a function that expects a string. Also, the function definitions and how to use them tend to be more clear. Especially when codebases grow larger, or when dealing with external libraries, it isn't always obvious which types are used where if it isn't explicitly defined.

So how does python deal with this?

## Python typing

By default, python does not require you to define the type when declaring a variable. However, Python 3 introduced type hinting in the [typing module](https://docs.python.org/3/library/typing.html). While this is great, it's actually nothing more than adding documentation to your code according to a predefined format. Let's look at the previous example again, but this time using type hints:

```
def addOne(number : int) -> int:
  return number + 1

print(addOne(5)) # prints '6'
print(addOne("5")) # TypeError: can only concatenate str (not "int") to str
```

The types of the variables are defined here, as well as the type of the return value. However, it didn't prevent anything. In fact, you can claim that the function takes in a String and returns a list. The code will still run, and the error will still be the same.

```
def addOne(number : str) -> list:
  return number + 1 
```
All these type hints are nothing but documentation, that's kept up to date by nothing more than good intentions. Anyone who has worked on large codebases know that documentation quickly becomes outdated, unless there's a strong culture of doing code reviews, or some kind of an automated check is in place, to match the documentation to the actual code.

## Linting

The only way to get true benefit from type hints, and to keep them up to date, is to introduce a linter that checks for them. A linter is a tool that performs static analysis on your code. In other words, it checks for errors without actually running the code. The first python linter that does type checking is [mypy](https://mypy-lang.org/). Let's take a look what it outputs for our previous example:

```
mypy example.py
example.py:5: error: Argument 1 to "addOne" has incompatible type "str"; expected "int"  [arg-type]
```

Exactly what we need! We can now add this linter to our CI/CD pipeline, and stop typing issues before they're even deployed. In conclusion, type hints are nice, but without a linter to enforce their correctness, a lot of their value is lost. 
