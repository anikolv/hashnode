---
title: "JVM compilers: Just in Time vs Ahead of Time"
seoTitle: "JVM compilers: Just in Time vs Ahead of Time"
seoDescription: "JVM compilers: Just in Time vs Ahead of Time"
datePublished: Sat Jun 10 2023 18:10:49 GMT+0000 (Coordinated Universal Time)
cuid: cliqbb9fu000409l39t6afuko
slug: jvm-compilers-just-in-time-vs-ahead-of-time
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1686420604962/84f8fc02-406c-4887-9918-ada2701d5a9a.png
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1686420633933/d1120929-058d-4fc4-a8a7-fa9e134abb5c.png
tags: java, compiler, jdk, jre, compilers

---

## What is a compiler and what does it do?

A compiler is a special type of software that translates source code written in one programming language into another language, usually machine code or "binary" that can be directly executed by a computer's processor.

The process of compiling involves several steps:

1. **Lexical Analysis**: The compiler breaks the source code down into its smallest units, known as tokens.
    
2. **Syntax Analysis**: The tokens are assembled into a parse tree, a data structure that represents the syntactic structure of the code according to the rules of the programming language.
    
3. **Semantic Analysis**: The compiler checks the parse tree for semantic errors, such as type mismatches or undeclared variables.
    
4. **Optimization**: The compiler optimizes the parse tree to improve the efficiency of the resulting machine code.
    
5. **Code Generation**: The compiler converts the optimized parse tree into machine code.
    

The main advantage of compilers is that they allow programs to run very efficiently. However, because the entire program must be compiled before it can be run, developers must correct all errors in their code before they can test it. This is in contrast to interpreted languages like **PHP, Ruby, Python, and JavaScript**, which can be run line-by-line, but are typically less efficient.

## A brief overview of the Java compilation process

The Java Development Kit (JDK) compiler, known as `javac`, is a key component of the Java programming environment. It's responsible for transforming Java source code into bytecode, an intermediate language that can be executed by the Java Virtual Machine (JVM).

Here's a brief overview of how `javac` works:

1. **Parsing**: The compiler reads the Java source code (.java files), breaks it into pieces according to the Java language syntax, and checks for syntax errors. This process generates a set of syntax trees.
    
2. **Type Checking**: The compiler checks the syntax trees for type errors. For example, it ensures that methods are called with the correct number and types of arguments, and that variables are used in ways that are consistent with their declared types.
    
3. **Bytecode Generation**: If no errors are found, the compiler generates Java bytecode instructions from the syntax trees. These instructions are stored in .class files, one for each class defined in the source code.
    
4. **Linking**: At runtime, the JVM links the classes together. This involves checking that all referenced classes can be found, and that they contain the methods and fields that the program expects.
    

The bytecode generated by `javac` is platform-independent, meaning it can be executed on any device that has a JVM. This is a key part of Java's "write once, run anywhere" philosophy. The JVM interprets or compiles the bytecode to machine code at runtime, allowing it to be executed by the computer's processor.

## Just-in-time compilation

Just-In-Time (JIT) compilation is a feature of the Java Virtual Machine (JVM) that increases the performance of Java applications at runtime. Here's how it works:

1. **Bytecode Interpretation**: When a Java program is run, the JVM starts by interpreting the bytecode, which is a platform-independent code compiled from the Java source code. The JVM executes the bytecode line by line, translating it into machine code that the computer's processor can understand. Machine code, also known as machine language, is the elemental language of computers. It is read by the computer's central processing unit ([CPU](https://www.techtarget.com/whatis/definition/processor)), is composed of digital [binary](https://www.techtarget.com/whatis/definition/binary) numbers and looks like a very long sequence of zeros and ones. Ultimately, the source code of every human-readable [programming language](https://www.techtarget.com/whatis/definition/programming-language-generations) must be translated to machine language by a compiler or an interpreter, because binary code is the only language that computer hardware can understand.
    
2. **HotSpot Identification**: As the JVM interprets the bytecode, it keeps track of the parts of the code that are executed frequently. These frequently executed parts are known as "hot spots".
    
3. **JIT Compilation**: Once a hot spot is identified, the JIT compiler kicks in. It compiles the entire hot spot into native machine code, which can be executed directly by the computer's processor. This compiled code is then cached for future use.
    
4. **Direct Execution**: The next time the JVM encounters the hot spot, it can skip the interpretation step and execute the compiled machine code directly. This is called warm-up. Warm-up is the time taken for the Java application to reach the optimum compiled code performance. It is the task of the Just-in-Time (JIT) compiler to deliver optimal performance by producing optimized compiled code from application bytecode. This greatly improves the performance of the program.
    

The JIT compiler is a part of the JVM. Most JVMs, including Oracle's HotSpot and OpenJDK, support JIT compilation. The JIT compiler is designed to optimize the execution speed of Java applications, making Java a competitive choice for high-performance computing.

## Ahead-of-time compilation

Ahead-Of-Time (AOT) compilation is a feature that allows Java bytecode to be compiled into native machine code **prior** to running the application. This is different from the traditional Just-In-Time (JIT) compilation, which compiles bytecode into machine code at runtime.

Here's how AOT compilation works in Java:

1. **Compilation**: Using an AOT compiler (such as the `jaotc` tool introduced in JDK 9), Java bytecode is compiled into native machine code before the application is run. This compilation process happens independently of the application's execution.
    
2. **Linking**: The resulting native machine code is linked with the JVM. The linking process ensures that the native code can interact correctly with the JVM during execution.
    
3. **Execution**: When the application is run, the JVM can directly execute the pre-compiled native code instead of interpreting and compiling the bytecode. This can lead to faster startup times and lower runtime overhead, as the JVM doesn't need to spend time interpreting the bytecode or compiling it to machine code during execution.
    

AOT compilation is particularly beneficial for short-lived applications or in environments where startup time is critical. However, it's worth noting that AOT compilation doesn't entirely replace JIT compilation. Instead, they complement each other. The JVM can still use JIT compilation for parts of the code that were not pre-compiled using AOT compilation.

GraalVM is a prominent JVM that supports AOT compilation. It includes a utility called `native-image` that can generate standalone native executables from Java applications. These executables include the application, necessary libraries, JDK, and a stripped-down JVM, which allows them to run without requiring a separate JVM installation.

## Side-by-side comparison

**Just-In-Time (JIT) Compilation**

*Pros:*

* **Runtime Optimization**: JIT compilers can optimize the program's performance based on actual runtime behaviour, leading to highly efficient execution for frequently used code paths.
    
* **Dynamic Adaptation**: JIT compilation can adapt to the system's current state, such as the available memory and the nature of the executing workload. This allows for more fine-tuned optimizations.
    
* **No Pre-Compilation Required**: With JIT, developers don't need to compile their code into machine language ahead of time. The JVM handles this at runtime.
    

*Cons:*

* **Startup Delay**: JIT compilation can lead to slower startup times because the JVM needs to interpret the bytecode and compile it to machine code during execution.
    
* **Memory Usage**: JIT compilers can use a significant amount of memory, especially for large applications, as they need to store the compiled code for quick execution.
    

**Ahead-Of-Time (AOT) Compilation**

*Pros:*

* **Fast Startup**: AOT compilation can lead to faster startup times because the JVM can skip the interpretation step and some parts of the JIT compilation.
    
* **Lower Runtime Overhead**: AOT compilation can reduce runtime overhead, as it doesn't require the storage of compiled code for repeated execution.
    
* **Predictable Performance**: Since the code is pre-compiled, the performance is more predictable and there are no "warm-up" effects that you might see with JIT compilation.
    

*Cons:*

* **Lack of Runtime Information**: AOT compilers lack runtime information, which can lead to less optimal code compared to JIT compilers.
    
* **Increased Binary Size**: AOT compilation can lead to larger binary sizes because it includes the entire compiled code.
    
* **Time-Consuming Compilation Process**: The process of AOT compilation can be time-consuming, especially for large applications.
    

## Conclusion

In conclusion, the choice between JIT and AOT will depend on the specific needs of your application. JIT is generally better for long-running applications where the initial startup time is less significant, while AOT can be beneficial for short-lived applications or in environments where startup time is critical, for example - use cases like AWS Lambda functions or other serverless computing models.

In serverless computing, functions are often short-lived and may need to start quickly in response to events. AOT compilation can help reduce startup latency, which is a critical performance factor in such environments.

By compiling the code into native machine code ahead of time, the JVM can skip the interpretation step and some parts of the JIT compilation, leading to faster startup times. This can be a significant advantage in serverless environments, where functions are frequently started and stopped.

Furthermore, AOT-compiled applications typically have a smaller memory footprint compared to JIT-compiled ones. This can lead to cost savings in serverless environments, where you often pay for the amount of memory used.

However, it's important to note that while AOT offers certain advantages, it also has its trade-offs, such as the lack of runtime optimization that JIT provides. Therefore, the choice between AOT and JIT should be made based on the specific requirements and constraints of your application and environment.