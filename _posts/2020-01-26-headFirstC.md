# Head First C

## Why C?

For a pretty long time, I had been trying to find a good book from which to learn the C programming language. As a scientist trained in physics and mathematics, I have always been interested in computational programming, and there is no modern language better than C to carry that out. For example, the `sklearn` and `scipy` libraries in Python are really well-documented and intuitive Python APIs wrapping C code, or have Cython pieces[^1]. Additionally, when I first learned programming, I learnt it in Python. The advantage is that Python allows you to just get started without having to worry about the nitty-gritty; it just abstracts all that for you. For example, since [PEP 237](https://www.python.org/dev/peps/pep-0237/), a Python `int` no longer overflows if the result cannot be held by a C `long int`, and instead its capacity is limited by the available memory, rendering it effectively unlimited precision. The programmer thus needn't worry about `int`, `short int`, `long int`, `long long int` and so on[^2].

Thus I wanted to learn more about what is going on under the hood, which I hope would overall make me a better programmer. To that end, I started learning more about C and hardware. I have started learning about hardware via **Inside The Machine** by Jon Stokes and watching some *assembly* game programming on Computerphile (yes, there is such thing).

## The Book

When I first saw the book I essentially judged it by its cover. I could not take it seriously, but I really am glad I decided to give it a shot.

## Interesting things I have learnt

As I previously discussed, I had already learnt some C/C++ prior to reading the book, so it was more to refine knowledge I had already gained, rather than start from scratch. Thus, I will include some particularly interesting and insightful things I have gained from the book. Nothing fundemental, just *ohhhh, sh#$%t* moments.

### String literals and copying

Consider the following piece of code, in which you have a string "123", and you want to cyclically permute it to "231"

```c
int main() {
    char* str = "123";
    char tmp = str[0];
    str[0] = str[1];
    str[1] = str[2];
    str[2] = tmp;
    
    return 0;
}
```

However, this code will lead to a memory error. On my machine, gcc compiles it, but running the resulting executable leads to `Bus Error: 10`. This is because a string literal such as "123" is stored in read-only memory (ROM). Thus, the `char*` pointer simply points to an address in ROM, which cannot be written to. If you want to modify a string declared to a literal, you need 

```c
char str[] = "123";
...
```

This will result in allocating 4 bytes of memory in the stack[^3], and copying the contents of the string literal unto the stack. Now, you can modify it. Thus, if you use `char*`, you are declaring a pointer to read-only memory, whereas declaring with `char str[]`, you copy its content to read-write memory in the stack. A bit tricky.

Alternatively, if you need to copy a string at run-time, *i.e.* dynamically, you cannot use the `char[]` declaration, because that requires the size to be a compile-time constant, and as we saw above, if you use `char*`, it is just a pointer to some other location in memory, perhaps ROM, so you do not own your data and it can be easily overwritten. Thus, C offers `strdup`. This has signature `char * strdup(const char* str)` and returns a pointer to the dynamically allocated memory on heap where a copy of `str` is made. Since it is on the heap, do not forget to de-allocate with `free`. But there you have it, that is how strings are copied in C, both on the stack and heap.

### Expressions have values

A C expression like `2+2` clearly has the value 4. By an expression, I mean a sequence of operators and the operands on which they act. Consider the following expression `x = 4`. Does it have a value? This assigns the `int x` to `4`, but the value of this expression is also `4`. So, we can write the following `y = (x = 4)`, and it assigns both `x` and `y` to `4`. A common use case of this idea is the two increment expressions `++x` and `x++`. They both increment `x`, however, these expressions have different *values*. The first expression increments `x` *then* returns its value, whereas the latter returns the value of `x` *then* increments, so if `x` is a pointer, `(++x) - (x++) == sizeof(*x)`.

### Pointer arithmetic is commutative and pointers decay

An interesting classic *gotcha* moment with C arrays is that a C array's name behaves like a pointer to the beginning of the array, and that random access into the array is implemented with pointer arithmetic. Combine this with commutativity of addition, and you get the following funny-looking piece of code

```c
int main() {
    int funny_c[] = {99, 105, 115, 102, 117, 110, 121};
    funny_c[3] == *(funny_c + 3);
    *(funny_c + 3) == *(3+funny_c);
    *(3+funny_c) == 3[funny_c];
    
    return 0;
}

```

Thus you can index using `funny_c[3]` or `3[funny_c]`, because subscription into an array is simply implemented with pointer arithmetic. Note that the expression `funny_c + 3` should be read as `funny_c + 3 * sizeof(*funny_c)`, so for a `char` pointer, it would be displaced by 3 bytes, whereas for an `int` pointer, it would be displaced by 12 bytes.

Another thing to keep in mind with arrays is their classic decay into pointers when passed into functions. They however are not actually themselves pointers: A pointer can be assigned to point elsewhere in memory, however an array does not accept assignment. Additionally, a pointer `p` itself exists as an object in memory, so if you take its address via `&p`, it will be an object living elsewhere in memory. For example

```c
#include <stdio.h>

int main() {
    int x = 5;
    int* p = &x;
    printf("pointer points to %p and lives at: %p\n", p, &p);
    int arr[] = {1,2,3};
    printf("arr points to %p, and lives at %p\n", arr, &arr);
    
    return 0;
}
```

On my system, I get the for the pointer `p` that "pointer points to 0x7ffeed4cda*08* and lives at: 0x7ffeed4cda*00*". However, for the array name, I get " "arr points to 0x7ffee5dcd9fc, and lives at 0x7ffee5dcd9fc". The array does not exist as a pointer variable elsewhere in memory. So, arrays decay to pointers when passed to functions, and their names can be used as pointers, but they are not pointers.

It is important to know that pointers are themselves objects, because then it is clear that pointers are passed to functions by value. So, when a pointer is passed into a function, the pointer itself is copied to the function stack, so if it is modified inside the function, *e.g.* `++p`, you do not lose access to your original data.

### Function Pointers

Just like an array name, a function name acts as a pointer to the function. For example

```c
int add_4(int x) {
    return x+4;
}

int main() {
    printf("the function add_4 lives at: %p\n", add_4);
    return 0;
}
```
prints "the function add_4 lives at: 0x1017b9f20". Technically speaking, you can indeed access the function by dereferencing the function pointer, however `(&add_4)(7) == add_4(7)`, so it is not necessary. Function pointers allow you to pass function as function arguments. When declaring a function parameter, you would declare for exmaple `int *f_ptr (int)`. Additionally, you can construct an array of function pointers. They must all have to have the same signature, *i.e.* same parameter types and return types. A typical declaration looks like `int (*funcptr_arr[3]) (int, int)`. Such a container is useful because it can make your program easily extensible and scalable, as opposed to a branch if/else or switch logic. Below we show an example, where we have an instruction consisting of two operands and an operation to perform on them.

```c

// an enum is represented internally as an int
// so ADD == 0, SUB == 1, MUL == 2, DIV == 3

typedef enum {
ADD,
SUB,
MUL,
DIV
} OP;

// an instruction consists of two operands and
// an operation to perform on them

typedef struct {
    int operand1;
    int operand2;
    OP op;
} instruction;

int add (int a, int b) {
    return a+b;
}

int sub (int a, int b) {
    return a - b;
}

int mul (int a, int b) {
    return a*b;
}

int div (int a, int b) {
    return a/b;
}
// declaring array of function pointers. They have to be homogeneous
// i.e. same argument and return types. THEY HAVE TO BE DECLARED IN
// THE SAME ORDER AS THE ENUM OPERATIONS
int (*ops[]) (int, int) = {add, sub, mul, div};

int carry_out(instruction inst) {
    // since enums are just ints, they can be used to index into
    // the function pointer array
    return ops[instruction.OP](inst.operand1, inst,operand2);
}

int main() {
    instruction inst1 = {5, 4, ADD};
    instruction inst2 = {9, 8, SUB};
    int res1 = carry_out(inst1); // == 9
    int res2 = carry_out(inst2); // == 1



```

Note that the above construction makes it easy to add additional instructions; you simply append an operation to the OP `enum` and a function pointer to the `ops` array of function pointers. For example, you can add a bitwise OR operation.

Function pointers make functions almost first-class citzens in C. They're still not quite first-class as in Python, due to the absence of closure in C, *i.e.* you cannot define a function inside another function like you do with other data types. It can only be defined in the global scope.


---



[^1]: Cython is a superset of the C programming language that allows combining C/C++ and Python code in the same source file. If you look at the source code of `scipy`, for example, `.pyx` source code is Cython. It can speed up Python code significantly via pre-compilation, static typing etc, and offers better memory management by using C types, but still allows access to your favorite Python objects such as dicts and lists. For more on Cython, click [here](www.cython.org)

[^2]: The first time I learnt about this was when comparing the performance of an implementation of the Fibonnaci sequence in Python to one in C. The former can return a Python `int`, whereas the latter would return a `double` or a `long long int`. If you declare the C version to return

[^3]: The fourth byte if for the null terminator '\0', with which c-strings always end.

[^4]: By that I mean they don't have to consist of valid C code, for example `#define WEIRD if (x<)` is a valid macro, in spite of not having balanced parentheses. Additionally, they don't have to end with semi-colons
