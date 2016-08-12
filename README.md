# Attacking the licensing problems with C++

From the early days of the commercialization of computer software, malicious programmers, also known as crackers have been continuously nettling the programmers of aforementioned software by constantly bypassing the clever licensing mechanisms they have implemented in their software, thus causing financial damages to the companies providing the software.

This trend has not changed in recent years, the more clever routines the programmers write, the more time is spent by the crackers to invalidate the newly created routines, and at the end the crackers always succeed. For the companies to be able to keep up with the constant pressure provided by the cracking community they would need to constantly change the licensing and identification algorithms, but in practice this is not a feasible way to deal with the problem.

An entire industry has evolved around software protection and licensing technologies, where renowned companies offer advanced (and expensive) solutions to tackle this problem. The protection schemes vary from using various resources such as hardware dongles, to network activation, from unique license keys to using complex encryption of personalized data, the list is long.

This article will provide a short introduction to illustrate a very simple and naive licensing algorithms' internal workings, we will show how to bypass it in an almost real life scenario, and finally present a software based approach to mitigate the real problem by hiding the license checking code in a layer of obfuscated operations generated by the C++ template metaprogramming framework which will make the life of the person wanting to crack the application a little bit harder. Certainly, if they are well determined, the code will also be cracked at some point, but at least we'll make it harder for them.

# A naive licensing algorithm

The naive licensing algorithm is a very simple implementation of checking the validity of a license associated with the name of the user who has purchased the associated software. It is NOT an industrial strength algorithm, it has just demonstrative power, while trying to provide insight on the actual responsibilities of a real licensing algorithm.

Since the license checking code is usually shipped with the software product in compiled form, I'll put in here both the generated code (in Intel x86 assembly) since that is what the crackers will see after a successful disassembly of the executable but also the C++ code for the licensing algorithm. In order to not to pollute the precious paper space with unintelligible binary code I will restrain myself to include only the relevant bits of the code, with regard to the parts which naively determines whether a supplied license is valid or not, together with the C++ code, which was used to generate the binary code.

The code was compiled using Microsoft Visual C++ (2015), in Release mode with optimization settings to favour a smaller executable, and I also have used the built in debugger of the VS IDE, to have a proper matching of the generated code with the source, to facilitate the educational aspect of this article.

The following is the source code of the licensing algorithm:

```cpp
static const char letters[] = "ABCDEFGHIJKLMNOPQRSTUVWXYZ";
bool check_license(const char* user, const char* users_license)
{
    std::string license;
    size_t ll = strlen(users_license);
    size_t l = strlen(user), lic_ctr = 0;
    int add = 0;

    for (size_t i = 0; i < ll; i++)
        if (users_license[i] != '-')
            license += users_license[i];

    while (lic_ctr < license.length() ) {
        size_t i = lic_ctr;
        i %= l;
        int current = 0;
        while (i < l) current += user[i ++];
        current += add;
        add++;
        if (license[lic_ctr] != letters[current % sizeof letters])
            return false;
        lic_ctr++;
    }
    return true;
}
```

The license which this method validates comes in the form of the following "ABCD-EFGH-IJKL-MNOP" and there is an associated `generate_license` method which will be presented as an appendix to this article.

Also, the naivety of this method is easily exposed by using the very proper name of `check_license` which immediately reveals to the up-to-be attacker where to look for the code checking the ... license. If you want to make harder for the attacker the identification of the license checking method I'd recommend either to use some irrelevant names or just strip all symbols from the executable as part of the release process.

The interesting part is the binary code of the method obtained via compilation of the corresponding C++ code. It is intentionally NOT the Debug version, since we hardly should ship debug version of the code to our customers.

```cpp
if (license[lic_ctr] != letters[current % sizeof letters])
    00FC15E4  lea         ecx,[license]  
    00FC15E7  cmovae      ecx,dword ptr [license]  
    00FC15EB  xor         edx,edx  
    00FC15ED  push        1Bh  
    00FC15EF  pop         esi  
    00FC15F0  div         eax,esi  
    00FC15F2  mov         eax,dword ptr [lic_ctr]  
    00FC15F5  mov         al,byte ptr [ecx+eax]  
    00FC15F8  cmp         al,byte ptr [edx+0FC42A4h]  
    00FC15FE  jne         check_license+0DEh (0FC1625h)  
    return false;
lic_ctr++;
    00FC1600  mov         eax,dword ptr [lic_ctr]  
    00FC1603  mov         ecx,dword ptr [add]  
    00FC1606  inc         eax  
    00FC1607  mov         dword ptr [lic_ctr],eax  
    00FC160A  cmp         eax,dword ptr [ebp-18h]  
    00FC160D  jb          check_license+7Fh (0FC15C6h)  
}
return true;
    00FC160F  mov         bl,1  
    00FC1611  push        0  
    00FC1613  push        1  
    00FC1615  lea         ecx,[license]  
    00FC1618  call        std::basic_string<char,std::char_traits<char>,std::allocator<char> >::_Tidy (0FC1944h)  
    00FC161D  mov         al,bl  
}
    00FC161F  call        _EH_epilog3_GS (0FC2F7Ch)  
    00FC1624  ret  
    00FC1625  xor         bl,bl  
    00FC1627  jmp         check_license+0CAh (0FC1611h)
```

Let's analyze it for a short while. The essence of the validity checking happens at the address `00FC15F8` where the comparison `cmp al, byte ptr [edx+0FC42A4h]` takes place (for those wondering, `edx` gets its value as being the remainder of the division at `00FC15F0`).

At this stage the value of the `al` register is already initialized with the value of `license[lic_ctr]` and that is the actual comparison to see that it matches the actually expected character. If it does not match, the code jumps to `0FC1625h` where the `bl` register is zeroed out (`xor bl, bl`) and from there the jump goes backward to `0FC1611h` to leave the method with the `ret` instruction found at `00FC1624`. Otherwise the loop continues.

The most common way of returning a value from a method call is to place the value in the `eax` register and let the calling code handle it, so before returning from the method the value of `al` is populated with the value of the `bl` register (via `mov al, bl` found at `00FC161D`).

Please remember, that if the check discussed before did not succeed, the value of the `bl` register was 0, but this `bl` was initialized to `1` (via `mov bl,1` at `00FC160F`) in case the entire loop was successfully completed.

If we think from the perspective of an attacker, the only thing that needs to be done is to replace in the executable the binary sequence of `xor bl,bl` with the binary code of `mov bl,1`. Since luckily these two have the same length (2 bytes) the crack is ready to be published within a few seconds.

Moreover, due to the simplicity of the implementation of the algorithm, a highly skilled cracker could easily create a key-generator for the application, which would be an even worse scenario, since the cracker didn't had to to modify the executable, thus further safety steps, such as integrity checks of the application would all be executed correctly, but there would be a publicly available key-generator which could be used by anyone to generate a license-key without ever paying for it, or malicious salesmen could generate counterfeit licenses which they could sell to unsuspecting customers.

Here comes in picture our C++ Obfuscating framework.

## The C++ Obfuscating framework

The C++ obfuscating framework provides a simple macro based mechanism combined with advanced C++ template meta-programming techniques for relevant methods and control structures to replace the basic C++ control structures and statements with highly obfuscated code which makes the reverse engineering of the product a complex and complicated procedure.

By using the framework the reverse engineering of the license checking algorithm presented in the previous paragraph would prove to be a highly challenging task due to the sheer amount of extra code generated by the frameworks engine.

The framework has adopted a familiar, BASIC like syntax to make the switch from real C++ source code the the macro language of the framework as easy and painless as possible.

### Functionality of the framework

The role of the obfuscating framework is to generate extra code, while providing functionality which is expected by the user, with as little as possible syntax changes to the language as could be achieved.

The following functionalities are provided by the framework:

 - wrap all values into a valueholder class thus hiding them from immediate
   access
 - providing a BASIC like syntax for the basic c++ control structures (if, for,
   while ...)
 - generating extra code to achieve complex code making it harder to understand
 - offering a randomization of constant values in order to hide the information

### Debugging with the framework

Like every developer who has been there, we know that debugging complex and highly templated c++ code sometimes can be a nightmare. In order to avoid this nightmare while using the framework we decided to implement a debugging mode.

In order to activate the debugging mode of the framework define the `OBF_DEBUG` identifier before including the obfuscation header file. Please see at the specific control structures how the debugging mode alters the behaviour of the macro.

### Using the framework

The basic usage of the framework boils down to including the header file providing the obfuscating functionality

```cpp
#include "instr.h"`
```

then using the macro pair `OBF_BEGIN` and `OBF_END` as delimiters of the code sequences that will be using obfuscated expressions.

For a more under the hood view of the framework, the `OBF_BEGIN` and `OBF_END` macros declare a `try`-`catch` block, which has support for returning values from the obfuscated current code sequence, and also provides support for basic control flow modifications such as the usage of `continue` and `break` emulator macros `CONTINUE` and `BREAK`.

#### Value and numerical wrappers

To achieve an extra layer of obfuscation, the numerical values can be wrapped in the macro `N()` and all numeric variables (`int`, `long`, ...) can be wrapped in the macro `V()` to provide an extra layer of obfuscation for doing the calculation operations. The `V()` value wrapper also can wrap individual array elements(`x[2]`), but not arrays (`x`) and also cannot wrap class instantiation values due to the fact that the macro expands to a reference holder object.

The implementation of the wrappers uses the link time random number generator provided by [Andrivet] and the values are obfuscated by performing various operations to hide the original value.

And here is an example for using the value and variable wrappers:

```cpp
int a, b = N(6);
V(a) = N(1);
```

After executing the statement above, the value of `a` will be 1.

The value wrappers implement a limited set of operations which you can use to change the value of the wrapped variable. These are the compound assignment operators: `+=`, `-=`, `*=`, `/=`, `%=`, `<<=`, `>>=`, `&=`, `|=`, `^=` and the post/pre-increment operations `--` and `++`. All of the binary operators `+`, `-`, `*`, `/`, `%`, `&`, `|`, `<<`, `>>` are also implemented so you can write `V(a) + N(1)` or `V(a) - V(b)`.

Also, the assignment operator to a specific type and from a different value wrapper is implemented, together with the comparison operators.

As the name implies, the value wrappers will wrap values by offering a behaviour similar to the usage of simple values, so be aware, that variables which are `const` values can be wrapped into the `V()` wrapper however as with real const variables, you cannot assign to them. So for example the following code will not compile:

```cpp
    const char* t = "ABC";
    if( V(t[1]) == 'B')
    {
        V( t[1] ) = 'D';
    }
```

And the following 

```cpp
    char* t = "ABC";
    if( V(t[1]) == 'B')
    {
        V( t[1] ) = 'D';
    }
```

will be undefined behaviour since the compiler highly probably will allocate the string `"ABC"` in a constant memory area. To work with this kind of data always use `char[]` instead of `char*`.

### Control structures of the framework

The basic control structures which are familiar from C++ are made available for immediate use by the developers by means of macros, which expand into complex templated code.

They are meant to provide the same functionality as the standard c++ keyword they are emulating, and if the framework is compiled in DEBUG mode, most of them actually expand to the c++ control structure itself.

#### Decision making

When there is a need in the application to take a decision based on the value of a specific expression, the obfuscated framework offers the familiar `if`-`then`-`else` statement for the developers in the form of the `IF`-`ELSE`-`ENDIF` construct.

##### The `IF` statement

For checking the true-ness of an expression the framework offers the `IF` macro which has the following form:

    IF (expression)
    ....statements
    ELSE
    ....other statements
    ENDIF

where the `ELSE` is not mandatory, but the `ENDIF` is, since it indicates the end of the `IF` blocks' statements.

And here is an example for the usage of the `IF` macro.

```cpp
IF( V(a) == N(9) )
     V(b) = a + N(5);
ELSE
     V(a) = N(9);
     V(b) = a + b;
ENDIF
```

Due to the way the `IF` macro is defined, it is not required to create a new scope between the `IF` and `ENDIF`, it is automatically defined and all variables declared in the statements between `IF` and `ENDIF` are destroyed.

Since the evaluation of the `expression` is bound to the execution of a hidden (well at least from the outer world) lambda unfortunately it is not possible to declare variables in the `expression` so the following expression:

    IF( int x = some_function() )

is not valid, and will yield a compiler error. This is partially intentional, since it gives that extra layer of obfuscation required to hide the operations done on a variable in a nameless lambda somewhere deep in the code.

In case the debugging mode is active, the `IF`-`ELSE`-`ENDIF` macros are defined to expand to the following statements:

```cpp
#define IF(x)  if(x) {
#define ELSE   } else {
#define ENDIF  }
```

#### Support for looping

There is a time when every application needs to iterate over a set of values, so I tried to re-implement the basic loop structures used in c++: The `for` loop, the `while` and the `do`-`while` have been reincarnated in the framework.

##### The `FOR` statement

The macro provided to imitate the `for` statement is:

    FOR(initializer, condition, incrementer)
    .... statements
    ENDFOR`

Please note, that since `FOR` is a macro, it should use `,` (comma) not the traditional `;` which is used in the standard C++ `for` loops, and do not forget to include your `initializer`, `condition` and `incrementer` in parentheses if they are expressions which have `,` (comma) in them.

The `FOR` loops should be ended with and `ENDFOR` statement to signal the end of the structure.

Here is a simple example for the `FOR` loop.

```cpp
FOR(V(a) = N(0), V(a) < N(10), V(a) += 1)
   std::cout << V(a) << std::endl;
ENDFOR
```

The same restriction concerning the variable declaration in the `initializer` as in the case of the `IF` applies for the FOR macro too, so it is not valid to write:

    FOR(int x=0; x<10; x++)

and the reasons are again the same as presented above.

In case of a debugging session the `FOR`-`ENDFOR` macros expand to the following:

```cpp
#define FOR(init,cond,inc) for(init;cond;inc) {
#define ENDFOR }
```

##### The `WHILE` loop

The macro provided as replacement for the `while` is:

    WHILE(condition)
    ....statements
    ENDWHILE

The while loop has the same characteristics as the `IF` construct and behaves the same way as you would expect from a well-mannered while statement: it checks the condition on the top, and executes the repeatedly the statements as long as the given condition is true.

Here is an example for the `WHILE`:

```cpp
    V(a) = 1;
    WHILE( V(a)  < N(10) )
        std::cout << "IN:" << a<< std::endl;
        V(a) += N(1);
    ENDWHILE
```

Unfortunately the `WHILE` loop also has the same restrictions as the `IF`: you cannot declare a variable in its condition.

In case the compilation is done in debugging mode, the `WHILE` evaluates to:

```cpp
#define WHILE(x) while(x) {
#define ENDWHILE }
```

##### The `REPEAT` - `UNTIL` construct

Due to the complexity of the solution, the familiar `do` - `while` construct of the C++ language had to be altered a bit, since the `WHILE` "keyword" was already taken for the benefit of the `while` loop, so I borrowed the `repeat` - `until` keywords to achieve this goal.

This is the syntax of the `REPEAT`-`UNTIL` construct:

    REPEAT
    ....statements
    UNTIL( expression )

This will execute the `statements` at least once, and then depending on the value of the `expression` either will continue the execution, or will stop and exit the loop. If the expression is `true` it will continue the execution from the beginning of the loop, if it is `false` it will stop the execution and exit the loop.

Please note, this is contradictory to the `repeat` - `until` logic offered by the Pascal language (which chose to continue the looping on a `false` expression), and it is similar the way the C++ language (and other similar languages) resolves the `do` - `while` looping.

And here is an example:

    REPEAT
        std::cout << a << std::endl;
        ++ V(a);
    UNTIL( V(a) != N(12) )

In case of debugging, the  `REPEAT`-`UNTIL` construct expands to the following:

```cpp
#define REPEAT   do {
#define UNTIL(x) } while (x);
```

#### Altering the control flow of the application

Sometimes there is a need to alter the execution flow of a loop, C++ has support for this operation by providing the `continue` and `break` statements. The framework offers the `CONTINUE` and `BREAK` macros to achieve this goal.

##### The `CONTINUE` statement

The `CONTINUE` statement will skip all statements that follow him in the body of the loop, thus altering the flow of the application.

Here is an example for the `CONTINUE` used in a `FOR` loop:

```cpp
FOR(a = 0, a < 5, a++)
   std::cout << "counter before=" << a << std::endl;
   IF(a == 2)
        CONTINUE
   ENDIF
   std::cout << "counter after=" << a << std::endl;
ENDFOR
```

and the equivalent `WHILE` loop:

```cpp
a = 0;
WHILE(a < 5)
    std::cout << "counter before=" << a << std::endl;
    IF(a == 2)
         a++;
         CONTINUE
    ENDIF
    std::cout << "counter after=" << a << std::endl;
    a++;
ENDFOR
```

Neither of these should print out the `counter after=2` text.

##### The `BREAK` statement

The `BREAK` statement terminates the loop statement it resides in and transfers execution to the statement immediately following the loop.

Here is an example for the `BREAK` statement used in a `FOR` loop:

```cpp
FOR(a = 0, a < 10, a++)
   std::cout << "counter=" << a << std::endl;
   IF(a == 1)
        BREAK
   ENDIF
ENDFOR
```

This loop will print `counter=0` and `counter=1` then it will leave the body of the loop, continuing the execution after the `ENDFOR`.

##### The `RETURN` statement

As expected, the `RETURN` statement returns the execution of the current function and will return the specified value to the caller function. Here is an example of returning 42 from a function:

```cpp
int some_fun()
{
OBF_BEGIN
    RETURN(42)
OBF_END
}
```

With the introduction of `RETURN`, an important issue arose: The obfuscation framework does not support the usage of `void` functions

##### The `CASE` statement

When programming in c++ the `switch`-`case` statement comes handy when there is a need to avoid long chains of `if` statements. The obfuscation framework provides a similar construct, although not exactly a functional and syntactical copy of the original `switch`-`case` construct.

Here is the `CASE` statement:

    CASE (<variable>)
        WHEN(<value>) [OR WHEN(<other_value>)] DO
        ....statements
        ....[BREAK]
        DONE
        [DEFAULT
        ....statements
        DONE]
    ENDCASE

The functionality is very similar to the well known `switch`-`case` construct, the main differences are:

1. It is possible to use non-numeric, non-constant values (variables and strings) for the `WHEN` due to the fact that all of the `CASE` statement is wrapped up in a templated, lambdaized well hidden from the outside world, construct. Be careful with this extra feature when using the debugging mode of the library because the `CASE` macro expands to the standard `case` keyword.
2. It is possible to have multiple conditions for a `WHEN` label joined together with `OR`.

The fall through behaviour of the `switch` construct which is familiar to c++ programmers was kept, so there is a need to put in a `BREAK` statement if you wish for the operation to stop after entering a branch.

And here is an example for the `CASE` statement:

```cpp
    std::string something = "D";
    std::string something_else = "D";
    CASE (something)
        WHEN("A") OR WHEN("B") DO
            std::cout <<"Hurra, something is " << something << std::endl;
            BREAK;
        DONE
        WHEN("C") DO
            std::cout <<"Too bad, something is " << something << std::endl;
            BREAK;
        DONE
        WHEN(something_else) DO
            std::cout <<"Interesting, something is " << something_else << std::endl;
            BREAK;
        DONE
        DEFAULT
            std::cout << "something is neither A, B or C, but:" << something <<std::endl;
        DONE
    ENDCASE
```
In case the framework is used in debugging mode the macros expand to the following statements:

```cpp
#define CASE(a) switch (a) {
#define ENDCASE }
#define WHEN(c) case c:
#define DO {
#define DONE }
#define OR
#define DEFAULT default:
```

# The naive licensing algorithm revisited

Now, that we are aware of a library that offers code obfuscation without too much headaches from our side (at least, this was the intention of the author) let's re-consider the implementation of the naive licensing algorithm using these new terms. So here it comes:

# Discommodities of the framework

Those who dislike the usage of CAPITAL letters in code may find the framework to be annoying. This is intentionally like this, because of the need to have familiar words that a developer instantly can connect to, and also to subscribe to the C++ rule, that macros should be uppercase.

This brings us back to the swampy area of C++ and macros. There are several voices whispering loudly that macros have nothing to do in a C++ code, and there are several voices echoing back that macros if wisely used can help C++ code as well as good old style C. I personally have nothing against the wise use of macros, indeed they came to be very helpful while developing this framework.

# References

[Andrivet] - Random Generator by Sebastien Andrivet - https://github.com/andrivet/ADVobfuscator

# license

The library is a header only library, released in the public domain under the MIT license.
