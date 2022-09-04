---
title: "Stanford-CS143 Lecture 02: Cool Overview"
linktitle: 'Lecture 02'
date: 2022-07-03
type: docs
draft: false

math: true

menu:
    stanford-compiler-lectures:
        parent: Contents
        weight: 2
---

Cool (Classroom Object Oriented Language).

5 programming assignments (all the 4 layers should be plug compatible)

* write a cool program
* lexical analysis
* parsing
* semantic analysis
* code generation

## Example: Hello World

```java
class Main {
    i : IO <- new IO;
    main():Int { { i.out_string("Hello World!\n"); 1; } };
};
```

A cool program (`*.cl`) should have class Main. A class contains several methods and attributes. Here `i` is an attribute with type IO. `main()` is a method with return value type `Int`. The method contains a block of statements enclosed in parentheses. Each statement should end with a semi-colon. Notice that we write `1;` instead of `return 1;`.

Some equivalent code snippets:

```java
class Main {
    i : IO <- new IO;
    main():IO { i.out_string("Hello World!\n") };
}
```

```java
class Main {
    i : IO <- new IO;
    main():Object { i.out_string("Hello World!\n") };
}
```

```java
class Main {
    main():Object { (new IO).out_string("Hello World\n") };
}
```

```java
class Main inherits IO {
    main():Object { self.out_string("Hello World\n") };
}
```

Here "inherit" means that class Main inherits all the attributes and methods of class IO. In the fourth code snippet, the "self" can be omitted.

## Example: Factorial

An example program that reads from stdin, adds 1 and writes to stdout:

```java
class Main inherits A2I{
    main() : Object {
        (new IO).out_string( i2a(a2i((new IO).in_string()) + 1).concat("\n") )
    };
};
```

New grammar: i2a() and a2i() in `atoi.cl` implements ascii-integer transformation. concat() can concatenate 2 strings together. If you want to compile a library, just list all the file names to the compiler.

A recursive version of `fact.cl`:

```java
class Main inherits A2I {
    
    main() : Object {
        (new IO).out_string( i2a(a2i(fact((new IO).in_string()))).concat("\n") )
    };
    
    fact(i: Int): Int {
        if (i = 0) then 1 else i * fact(i-1) fi
    };
};
```

New grammar: `if-then-else-fi` structure.

An iterative version of `fact.cl`:

```java
class Main inherits A2I {
    
    main() : Object {
        (new IO).out_string( i2a(a2i(fact((new IO).in_string()))).concat("\n") )
    };
    
    fact(i: Int): Int {
        let fact: Int <- 1 in {
            while (not (i = 0)) loop
                {
                    fact <- fact * i;
                    i <- i - 1;
                }
            pool;
            fact;
        }
    };
};
```

New grammar: `while-loop-pool` structure, `let` to define a local variable.

Pay attention that `=` in cool is a comparison operator, assignment operator should be `<-` !

## Example: list

```java
class List {
    item: String;
    next: List;
    
    init(i: String, n: List): List {
        {
            item <- i;
            next <- n;
            self;
        }
    };
    
    flatten(): String {
        if (isvoid next) then
            item
        else
            item.concat(next.flatten())
        fi
    };
};

class Main inherits IO {
    main(): Object {
        let hello: String <- "Hello ",
            world: String <- "World!",
            newline: String <- "\n",
            nil: List,
            list: List <- 
                  (new List).init(hello,
                      (new List).init(world,
                          (new List).init(newline, nil)))
        in
            out_string(list.flatten())
    };
};
```

New grammar: define multiple variables at the same time, the "isvoid" function.

A more sophisticated flatten() function:

```java
class List {
    item: String;
    next: List;
    
    init(i: String, n: List): List {
        {
            item <- i;
            next <- n;
            self;
        }
    };
    
    flatten(): String {
        let string: String <-
            case item of
                i: Int => i2a(i);
                s: String => s;
                o: Object => { abort(); ""; };
            esac
        in
            if (isvoid next) then
                string
            else
                string.concat(next.flatten())
            fi
    };
};
```

New grammar: `case of - esac` structure, abort() function

Note: the `""` is used to convince the type checker that the return value is of type String.

