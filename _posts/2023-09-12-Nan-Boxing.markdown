---
layout: post
title:  "Dynamic Typing and NaN Boxing"
date:   2023-09-12
---

I’m making a Lisp.  And I'm checking it twice. My Lisp, like most Lisp dialects, is *dynamically-typed*.  There are many ways to define dynamic typing. But for the purposes of this post the following definition will be the most helpful: **in a dynamically-typed language type checking happens at runtime**. Dynamically typed languages benefit from more flexible code than their statically typed counterparts. Variables don’t require type declarations and they can be reassigned data of different types, at different stages, in the same execution scope. And in dynamically typed languages functions can accept and return any type of data.

If you type check at runtime you need a way to store values and type information while the program is being evaluated.  The obvious way to store this information is out of band; that is, store the value and the type information separately. But the value will need to be associated with the type somehow and the most common solutions to this problem have disadvantages (described below) when it comes to memory usage and speed.  In this post I’ll describe a clever optimization called *NaN Boxing* that addresses these issues by storing the value and the type together in one 64 bit representation.

"NaN" is an abbreviation for "not a number". Most programming languages support [IEEE 754 double precision floating point numbers](https://en.wikipedia.org/wiki/Double-precision_floating-point_format) (*doubles*), which is a format with well-defined operations supported directly on modern CPUs. NaN is a special value in doubles that usually turns up when you do operations the CPU wasn't prepared for, like getting the square root of -1.

NaN Boxing works by storing data in the unused bits of NaN. By the way, NaN has unused bits. The idea is that every value can be stored in the space of a double. Valid doubles can be read directly and all the other types have their data stuffed into those fallow NaN bits I just mentioned. Before I go into further detail it will be helpful to know a little more about how doubles are represented internally.

# The anatomy of a double

A double is an 8 byte (64 bit) representation of a floating point number. Sections of the bits of a double are designated to represent different aspects of the floating point number. I hereby submit my contribution to the already voluminous corpus of illustrations of IEEE 754 standard for representing a double precision floating-point number. Behold! 

<figure class="highlight">
    <pre>
        <div style="display: flex; align-items: center;">
            <div class="custom_wrapper">
                <div class="sign_label"><span>sign</span></div>
                <div class="exponent_label"><span>exponent</span></div>
                <div class="mantissa_label"><span>mantissa</span></div>
                <div class="bits container_1"><span>s</span></div>
                <div class="bits container_2"><span>eeeeeee</span></div>
                <div class="bits container_2"><span>eeee<span class = "mantissa">mmmm</span></span></div>
                <div class="bits container_3"><span>mmmmmmmm</span></div>
                <div class="bits container_3"><span>mmmmmmmm</span></div>
                <div class="bits container_3"><span>mmmmmmmm</span></div>
                <div class="bits container_3"><span>mmmmmmmm</span></div>
                <div class="bits container_3"><span>mmmmmmmm</span></div>
                <div class="bits container_3"><span>mmmmmmmm</span></div>
                <div class="bits formula"><span> = (-1)<sup><span class="sign">s</span></sup> * <span class="mantissa">m</span><sup><span class="exponent">e</span></sup></span></div>
            </div>
        </div>
    </pre>
</figure>

Going from left to right, the first bit (dubed, normatively, the *most significant bit*) is the  <span class="sign">*sign bit*</span>, which signals weather the number is positive or negative. The next 11 bits are the <span class="exponent">*exponent bits*</span>. The exponent is [biased]( https://en.wikipedia.org/wiki/Exponent_bias). This is both a standard for encoding exponents with either negative or positive values and a slanderous accusation about the exponent’s character.  An 11 bit biased exponent can represent a number between −1022 and +1023.  The remaining 52 bits are the <span class="mantissa">*mantissa*</span>. And for those of us interested in optimizing run time type checking, that’s where the magic happens.

Some examples of doubles:
<figure class="highlight">
    <pre>
    <span><span class="sign">0</span> <span class="exponent">01111111111</span> <span class="mantissa">0000000000000000000000000000000000000000000000000000</span> = 1</span>
    <span><span class="sign">1</span> <span class="exponent">10000000000</span> <span class="mantissa">0000000000000000000000000000000000000000000000000000</span> = -2</span>
    <span><span class="sign">0</span> <span class="exponent">01111111000</span> <span class="mantissa">0101101111110000101010000111010000100111111100000001</span> = 2.7182818</span>
    <span><span class="sign">0</span> <span class="exponent">11111111111</span> <span class="mantissa">1000000000000000000000000000000000000000000000000000</span> = NaN</span>
    <span><span class="sign">0</span> <span class="exponent">11111111111</span> <span class="mantissa">1000000000000000100000000000000000100000000000000000</span> = NaN</span>
    <span><span class="sign">1</span> <span class="exponent">11111111111</span> <span class="mantissa">0101101111110000101010000111010000100111111100000001</span> = NaN</span>
</pre>
</figure>
As you can see from the last three examples NaN can be encoded in many different ways. Any double whose exponent bits are all 1s is treated as Not a Number.

There are actually two kinds of NaNs.  If the most significant bit of the mantissa is 0, then this is a signaling NaN (*sNaN*). IEEE 754 specifies that the CPU may [raise an exception](https://www.geeksforgeeks.org/interrupts-and-exceptions/) if an sNaN is used in certain operations. But if the most significant mantissa bit is a 1, wellsir you’ve got yourself a quiet NaN (*qNaN*).  qNaNs are produced when the result of an operation is not a number. And, most importantly for NaN boxing, qNaNs do not raise an exception, so we are free to (ab)use them without causing our program to crash.

IEEE 754 suggests that the mantissa of a qNaN should, [*by means left to the implementer’s discretion*](https://iremi.univ-reunion.fr/IMG/pdf/ieee-754-2008.pdf#page=46), contain diagnostic information. We can instead use the mantissa to pass around our values and type information.

All the code examples below are written in C, since that’s the language in which I’m implementing my Lisp. I’m going to assume the reader has a little familiarity with that language and with the concept of [bitmasking](https://en.wikipedia.org/wiki/Mask_(computing)).



# Why would anyone want to touch a NaN’s bits?

One way to store type information and values together is to use a *tagged union*. Consider the following example:

```c
struct Expression {
    enum { TYPE_FLOAT, TYPE_INT, TYPE_STRING } tag;
    union {
        double as_float;
        char *as_string; 
        int as_int; 
 }  value;
}
```
An *Expression* is a struct with two fields: `tag` and `value`. The `value` field is a *union*. A union is a special data type that overlays all its members in one location in memory. The enum `tag` is meant to signal which member of the union is actually stored in the `value` field.

Then, a tagged union is a special structure with a value field, which could have one of several different fixed types, and a tag field that indicates which type is in use. For instance, an Expression of type float is created like this:

```c
struct Expression *e = malloc(sizeof Expression);// declare e as a pointer to an Expression, allocate enough heap memory to sore e
e->tag = TYPE_FLOAT;// tag e as a float
e->value.as_float = 2.71828;// store a value in e as a float
```
We declare a pointer `e` to an Expression, then tag it as a float, store a value in the appropriate member of the union and then release the Expression back into the wild. 

Operations on Expressions can then switch on the tag:
```c
void print_expression(Expression *e)
{
    switch(e->tag)
    {
        case TYPE_FLOAT :{
            /* print a float */
	        printf(“%f”, e->value.as_float);
        break;
        }
        case TYPE_INT :{
            /* print an int */
	        printf(“%d”, e->value.as_int);
        break;
        }
        case TYPE_STRING :{
            /* print a string */
	        printf(“%s”, e->value.as_string);
        break;
        }
       default :{
            printf(“unknown type”);
	    break;
       }
    }
}
```
Without the tag we wouldn’t be able to print the value safely because we wouldn’t know which of the three possible types of data was actually stored at the `value` address.

Now to the disadvantages I mentioned in the introduction: Obviously the `tag` and the `value` fields of an Expression both occupy space. Further, a union must allocate [enough memory for its largest member](https://www.geeksforgeeks.org/c-unions/). So an Expression is going to be larger than the 64 bits needed for a double or for a pointer (the `value` field’s two largest members) regardless of what value is actually stored in the `value` field. And, therefore, an  Expression represented as a tagged union will be larger than a NaN. A more compact representation would mean that more Expressions fit into the [cache line](https://lemire.me/blog/2022/06/06/data-structure-size-and-cache-line-accesses/) and that affects speed.

Another disadvantage to storing values and type information with structs of any kind is that structs tend to be passed by reference and stored in heap memory.  In the example I gave above the first line allocates some heap memory and returns a pointer. Referencing Expressions this way will lead to [cache misses](https://redis.com/glossary/cache-miss/), which also affect speed. 

NaN boxing addresses these issues and you get the illicit thrill of breaking IEEE protocol.


# Implementation

Recall that the plan is to stuff extra values into the mantissa bits of a qNaN. We have 51 mantissa bits available. The most significant 3 available bits will be reserved for the tag. This gives us 48 mantissa bits to store value information. I’ll refer to these bits as the *payload*. Behold!

<figure class="highlight">
    <pre>
        <div style="display: flex; align-items: center;">
            <div class="custom_wrapper">
                <div class="sign_label"><span>sign</span></div>
                <div class="exponent_label"><span>exponent</span></div>
                <div class="mantissa_label"><span>payload</span></div>
                <div class="q_bit_label"><span>signal</span></div>
                <div class = "tag_label"><span>tag</span></div>
                <div class="bits container_1"><span>s</span></div>
                <div class="bits container_2"><span>1111111</span></div>
                <div class="bits container_2"><span>1111<span class = "q_bit">1</span><span class = "tag">ttt</span></span></div>
                <div class="bits container_3"><span>pppppppp</span></div>
                <div class="bits container_3"><span>pppppppp</span></div>
                <div class="bits container_3"><span>pppppppp</span></div>
                <div class="bits container_3"><span>pppppppp</span></div>
                <div class="bits container_3"><span>pppppppp</span></div>
                <div class="bits container_3"><span>pppppppp</span></div>
                <div class="bits formula"><span> = tagged NaN</span></div>
            </div>
        </div>
    </pre>
</figure>

Data can be encoded and retrieved through bit masking. We need masks for important NaN segments:

```c
#define MASK_QNAN 0x7ff8000000000000 //0111111111111000000000000000000000000000000000000000000000000000
#define MASK_PAYLOAD 0x0000ffffffffffff //0000000000000000111111111111111111111111111111111111111111111111
#define MASK_TAG 0x0007000000000000 //0000000000000111000000000000000000000000000000000000000000000000
#define MASK_EXPONENT 0x7ff0000000000000 //0111111111110000000000000000000000000000000000000000000000000000
```
Encoding is done by functions that take the original value and use bitwise-or to build a tagged NaN with the original value encoded into the payload

```c
double encode_value(value val)
{ 
    return MASK_QNAN | MASK_VALUE_TYPE | (uint64_t)val; 
}
```
We need maks for our types:
```c
#define MASK_INT 0x0001000000000000 //0000000000000001000000000000000000000000000000000000000000000000
#define MASK_STRING 0x0002000000000000 //0000000000000010000000000000000000000000000000000000000000000000
```
We don’t need a mask for floats because any non-NaN is a float.  We get them for free!

Decoding is done by functions that take a tagged NaN and use bitwise-and to isolate the payload and cast it to the expected type.
```c
value decode_value(double tagged_nan)//notice this function returns the made up type *value*.
{
    return tagged_nan & MASK_PAYLOAD; 
}
```
# Encode and decode ints
We can store an int directly in the payload. Even though the payload is 48 bits, I am going to use 32 bit ints because these are well defined in IEEE and supported in C.  So we will need a special mask for the 32 bit payload. 
```c
#define MASK_PAYLOAD_INT 0x00000000ffffffff //0000000000000000000000000000000011111111111111111111111111111111
```
On line 3 of the *encode_int* function I am casting the value to a 32 bit int before casting to a 64 bit int. This is to avoid an operation called [sign extension](https://en.wikipedia.org/wiki/Sign_extension#:~:text=Sign%20extension%20(abbreviated%20as%20sext,positive%2Fnegative)%20and%20value) that is meant to preserve the sign of a binary number when increasing its size. Under sign extension the most significant bit of the original number is extended to fill the new significant bits of the extended number. For instance this 16 bit integer `1101000000000100` would be sign-extended to this 32 bit integer `11111111111111111101000000000100`. Obviously this would be a problem.

The strange type casting one line 5 is an example of a technique called [type punning](https://en.wikipedia.org/wiki/Type_punning). In this case it ensures that the compiler will read the raw 64 bits of the encoded NaN without any conversion shenanigans.
```c
double encode_int(double value)
{
   uint32_t narrowed = value;//avoid sign extension
   uint64_t boxed = MASK_QNAN | MASK_INT | (uint64_t)narrowed;
      return *(double *)&boxed;//type punning. This says, roughly, treat “boxed” as a pointer, then cast it as a pointer to a double, then dereference that pointer
}
```
Then decoding an int just involves some more type punning—so the compiler will read the bare 64 bits—masking the 32 bits off the payload and returning the truncated 32 bits of the result.
```c
int32_t decode_int(double value)//notice this function returns an int32_t
{
   return *(uint64_t *)&value & MASK_PAYLOAD_INT;
}
```
# Encode and decode strings
Encoding strings is slightly more involved than encoding ints because we can’t store a string directly in the payload. But we can store a *string pointer* in the payload. You might be worried that the payload isn't big enough. After all, on a 64 bit architecture pointers are 64 bits long. True. But 64 bits is a huge address space and modern 64 bit architectures only use the lower 48 bits of a pointer and ignore the rest. So we have just enough payload to store the meaningful portion of a pointer.

We’ll need two functions for encoding a string. One function takes a string pointer and returns a double. This function will allocate heap memory for the string and the terminating character, copy the string to the new address and pass the new address to the string encoding function.
```c
double new_string(char *str)
{
   char *new_address = malloc(strlen(str) + 1);
   strcpy(new_address, str);
   return encode_string(new_address);
}
```
From here the encoding function takes a char pointer and returns a double. It works pretty much the same way as *encode_int*.
```c
double encode_string(char *str)
{
    uint64_t encoded = MASK_QNAN | MASK_STRING | (uint64_t)str;
    return *(double *)&encoded;
}
```
Decoding a string works the same as decoding an int, but we use the 48-bit mask.
```c
char *decode_string(double value)//notice this function returns a char pointer
{
   return *(uint64_t *)&value & MASK_PAYLOAD;
}
```
# Printing Values

Our previous enum *tag* can be re-define as a type.
```c
typedef enum Tag { 
        TYPE_FLOAT, 
        TYPE_INT, 
        TYPE_STRING,
        TYPE_ERROR //I’ve added an error type
        } Tag;
```
Then we can use it in a helper function that takes an encoded value and returns the type.
```c
Tag get_type(double value)
{
  uint64_t converted = *(uint64_t *)&value;
  uint64_t type = converted & MASK_TAG;
  /* if the value is NOT NaN then it is a float*/
  if ((~converted & MASK_EXPONENT) != 0)
    return TYPE_FLOAT;
  switch (type)
  {
  case (MASK_QNAN | MASK_INT):
    return TYPE_INT;
  case (MASK_QNAN | MASK_STRING):
    return TYPE_STRING;
  }
  return TYPE_ERROR;//this is where the new error type comes in handy
}
```
Finally, our previous print function only needs some slight adjustments:
```c
void print_value(double value)// print_expression changed to print_value; *e changed to value
{
    switch(get_type(value))// e->tag changed to get_type(value)
    {
        case TYPE_FLOAT :{
            /* print a float */
	        printf(“%f”, value);// no need to decode non-NaN floats. We get them for free.                      
        break;
        }
        case TYPE_INT :{
            /* print an int */
	        printf(“%d”, decode_int(value)); // e->as_int changed to decode_int(value)
        break;
        }
        case TYPE_STRING :{
            /* print a string */
	        printf(“%s”, decode_string(value));// e->as_string  changed to decode_string(value)
        break;
        }
        case TYPE_ERROR :{
            printf(“unknown type”);
	    break;
       }
    }
}
```
# Conclusion

I’ve created a [repo for this project](https://github.com/AveryBurke/nan_boxing) that implements a simple repl. The repl makes a roundtrip from parsing input, to encoding the parsed input into a tagged nan, to decoding the parsed input and printing the result. 

```lisp
    $> (+ 8 -8 .1234)
    type: string, value: (
    type: string, value: +
    type: int, value: 8
    type: int, value: -8
    type: float, value: 0.123400
    type: string, value: )
```
This uses all the encoding functions, decoding functions and bitmasks described in this post. The only difference is the addition of the repl and the parser, and the extra information in the print output. 

In the full Lips the `string` type will be replaced with a `symbol` type and `print_value` will be replaced with an `eval` function that looks up the symbol definitions in an environment.

# Further Reading

I’m working on this lisp during my time at the [Recurse Center](https://www.recurse.com/) and, as seems to happen to me quite often, I just learned that another Recurser got to this topic before I did. So for a much more thorough treatment read: [The Secret Life of NaN, by Annie Cherkaev](https://anniecherkaev.com/the-secret-life-of-nan)

Robert Nystrom has a [section on NaN boxing in the final chapter of Crafting Interpreters](https://craftinginterpreters.com/optimization.html#nan-boxing)

I first came across this topic while reading [this article](https://github.com/Robert-van-Engelen/tinylisp/blob/main/tinylisp.pdf)



