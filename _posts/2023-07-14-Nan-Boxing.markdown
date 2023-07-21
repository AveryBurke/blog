---
layout: post
title:  "Nan Boxing"
date:   2023-07-14
---

I’m making a Lisp. AndImcheckingittwice. And Lisp, like JavaScript, is a *dynamically typed* language. This means the type of a variable is associated with the value bound to the variable rather than with the variable itself. As opposed to a *statically typed* language, where the type of a variable is associated with the variable, rather than with the value.  One way to think about this distinction is that in dynamically typed languages a variable is simply a name used to reference some value; whereas in a statically typed language a variable is a compartment used to store values of a particular type—when you declare a variable ```x```, say, *as an int* you set the type of the variable so that only ints can be stored in the compartment labeled ```x```.

One implication of these strategies is that type checking happens at runtime in dynamically typed languages and before runtime in statically typed languages. Dynamically typed languages must check the types of variables at runtime, because the types can’t be known until the values are known.  

NaN Boxing is an optimization (a hack really) for runtime type checking.  It works by storing data in the unused bits of an [IEE 754 double precession NaN]( https://en.wikipedia.org/wiki/NaN). By the way, NaNs have unused bits. Unsettling, I know.  The idea is that every value can be a double and every NaN can store the type of a value and a pointer to that value. Lua implementations do it. JavaScript implementations do it. Let’s do it! Let’s store pointers and type information in the mantissa of a NaN! Don’t know what that means? I’ll explain.

# The anatomy of a NaN

A NaN is a special case of a floating-point number. So first we should review the anatomy of floating-point numbers. In this case I’m only going to consider doubles.

# The anatomy of a double

A double is an 8 byte (64 bit) representation of a floating point number.  I hereby submit my contribution to the already voluminous corpus of illustrations of [IEEE 754 standard for representing a double precision floating-point number](https://en.wikipedia.org/wiki/Double-precision_floating-point_format): 

<figure class="highlight">
    <pre>
        <div style="display: flex; align-items: center;">
            <div class="custom_wrapper">
                <div class="sign_label"><span>sign</span></div>
                <div class="exponent_label"><span>exponent</span></div>
                <div class="mantissa_label"><span>mantissa</span></div>
                <div class="bits container_1"><span>s</span></div>
                <div class="bits container_2"><span>eeeeeee</span></div>
                <div class="bits container_2"><span>eeeeeeee</span></div>
                <div class="bits container_2"><span>eeeeeeee</span></div>
                <div class="bits container_3"><span>mmmmmmmm</span></div>
                <div class="bits container_3"><span>mmmmmmmm</span></div>
                <div class="bits container_3"><span>mmmmmmmm</span></div>
                <div class="bits container_3"><span>mmmmmmmm</span></div>
                <div class="bits formula"><span> = (-1)<sup><span class="sign">s</span></sup> * <span class="mantissa">m</span><sup><span class="exponent">e</span></sup></span></div>
            </div>
        </div>
    </pre>
</figure>

Going from left to right, the first bit is the <span class="sign">*sign bit*</span>, which signals weather the number is positive or negative. The next 11 bits are the <span class="exponent">*exponent bits*</span>. The exponent is [biased]( https://en.wikipedia.org/wiki/Exponent_bias). This is both an ad hominem attack against the exponent and a standard for encoding exponents with either negative or positive values.  An 11 bit biased exponent can represent a number between −1022 and +1023, but is should be never be allowed to server on a jury.  The remaining 52 bits are the <span class="mantissa">*mantissa*</span>. And for those of us interested in optimizing run time type checking, that’s where the magic happens.

Some examples of doubles:
<figure class="highlight">
    <pre>
    <span><span class="sign">0</span> <span class="exponent">01111111111</span> <span class="mantissa">0000000000000000000000000000000000000000000000000000</span> = 1</span>
    <span><span class="sign">1</span> <span class="exponent">10000000000</span> <span class="mantissa">0000000000000000000000000000000000000000000000000000</span> = -2</span>
    <span><span class="sign">0</span> <span class="exponent">01111111000</span> <span class="mantissa">1000000000000000000000000000000000000000000000000000</span> = 0.01171875</span>
    <span><span class="sign">0</span> <span class="exponent">111111111</span> <span class="mantissa">1000000000000000000000000000000000000000000000000000</span> = NaN</span>
    <span><span class="sign">0</span> <span class="exponent">111111111</span> <span class="mantissa">1000000000000000100000000000000000100000000000000000</span> = NaN</span>
    <span><span class="sign">1</span> <span class="exponent">111111111</span> <span class="mantissa">1111100000000000000000000000000000000000000000000000</span> = NaN</span>
</pre>
</figure>


As you can see from these examples NaNs are not unique.  Any double whose exponent bits are all 1s is treated as a NaN.  

If the first bit of the mantissa is a 0, then this is a signaling NaN (sNaN) and it may raise an exception. Signaling NaNs (sNaNs) can be used as the default values of uninitialized variables. This ensures that using such a variable in an expression will raise an exception 

If the first mantissa bit is a 1, wellsir you’ve got yourself a quite NaN (qNaN, the pilled NaN). qNaNs are more common. They are produced when the result of an operation is not a number—dividing by zero, for instance.  But the important feature for NaN boxing, is that qNaNs do not rase an exception. It’s always the quite ones.

IEE 754 suggests that the mantissa of a qNaN should, *by means left to the implementer’s discretion*, contain diagnostic information about the cause of the NaN.  But we’re the implementer now, dog.  We can instead use the mantissa to pass around type information and pointers.  Before I give an example of how one might implement NaN boxing, I am going to provide a motivation for *why* one might want this solution over any other.

# Why would anyone want to touch a NaN’s bits?

In C you declare a variable like so ```int x;```.  The *type declaration* ```int``` tells the complier to reserve enough memory for an ```int``` and the address of that memory block is bound to the variable.  The type declaration also tells the complier to read x *as an int*. One advantage of doing things this way is that the complier can catch a type error.  If you try to add ```x``` to a string ```char *s  = “  vs. Sever”; return x + s;``` the complier will tell you you’ve done something wrong, because you've told it ahead of time how to read each varaible.

Now, in JavaScript adding a number to a string will, perversely, not throw an error. But the interpreter still needs to know the type of ```x``` and the type of ```s``` in order to know how combine them and, in classic JS fashion, return the least intuitive result.  But there’s no type declarations in JS. That is, the type, again, is not associated with the variable; it’s associated with the value. So, the parser stores the type information with the value. Then decisions about how to best confuse the hapless developer are made when the expression is evaluated.

The point I’m trying to illustrate, aside from venting about JS, is that dynamically typed languages must store values with some structure. My lisp implementation is no different. When parsing an expression each resulting token must at least have information about the type of the value and the value itself. One way to implement this kind of structure is with a *tagged union*. An example in C:

```c
struct Expr {
    enum { TYPE_FLOAT, TYPE_BOOL, TYPE_SYM } tag;
    union {
        double as_float; /* TYPE_FLOAT: double precision float number */
        char *as_symbol; /* TYPE_SYM: pointer to the symbol name */
        int as_bool; /* TYPE_BOOL: a 1 == #t or 0 == #f */
    } value;
};
```

An ``` Expr``` is a ```struct``` with two fields: ```type``` and ```value```. The values field is a ```union```. A ```union``` overlays all its members in one location in memory.  The enum ```tag``` is meant to signal which member of the union is actually stored in the ```value``` feild.  A number might be declared like this:
```c
struct Expr *e = malloc(sizeof Expr); /* declare e as a pointer to an Expr, create enough memory to sore e */
e->tag = TYPE_FLOAT; /* tag e as a float*/
e->value = 12.5; /* store only the best floating point numbers */
```
We create a pointer ```e``` to an Expr, then tag it as a float, store a value in it and then release it back into the wild. Then evaluation can happen inside a big switch statement: 

```c
some_value eval_expr(Expr *e)
{
    switch(e->tag)
    {
        case TYPE_FLOAT :{
            /* evaluate e->value as a float */
            break;
        }
        case TYPE_BOOL :{
            /* evaluate e->value as a bool*/
            break;
        }
        case TYPE_SYM :{
            /* evaluate e->value as a symbol */
            break;
        }
        /* add more cases as more types are added to the language */
    }
}
```

A tagged union is more memory efficient than, say, a ```struct``` because the ```values``` field only occupies one location in memory, as opposed to a contiguous block in memory (as do the fields of a ```struct```). But the ```values``` field needs to be large enough to store its largest member. Which is an 8 byte double. That means that 8 bytes are used to store a 1 byte bool. Inefficient! Not only that, a new ```Expr``` is created for every value, even duplicate values. For a large amount of code, this can start to use up a lot of space. And, not for nothing, but ```Expr```s are passed as pointers. So, evaluating a double requires jumping to a new location on the heap.  NaN boxing addresses all these issues and you get the illicit thrill of break IEE protocol to boot! 

