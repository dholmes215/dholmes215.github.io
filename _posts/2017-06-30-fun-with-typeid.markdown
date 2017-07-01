---
layout: post
title:  "Fun with typeid()"
date:   2017-06-30 22:52:00 -0500
categories: c++
---
My organization's product has its origins in a dark time and place where C++ implementations were lacking not only standards compliance but also basic features.  In particular, we had no runtime time information (RTTI), which meant no `dynamic_cast` and no `typeid()`.

In lieu of RTTI language support, we did have an accidental homebrew form of RTTI where each subclass derived of a certain type had a pointer to an object that was unique to either its type or some intermediate base type that we were interested in casting to. In effect:

{% highlight cpp %}

    int typeA = 1;
    int typeB = 2;

    class Base {
    public:
        Base(int * iType) : type(iType) { };
        virtual int * GetType() const { return type; };
        int * type;
    };

    class A : public Base {
    public:
        A() : Base(&typeA) { };
    };

    class B : public Base {
    public:
        B() : Base(&typeB) { };
    };
{% endhighlight %}

For other legacy code reasons, we also often find ourselves processing lists of pointers to objects of various polymorphic types, of which we only want to process entries of a certain type. 


{% highlight cpp %}

    int count = 0;
    for (auto ptr : ptrArray) {
        if (ptr->GetType() == &typeA) {
            count++;
        }
    }
{% endhighlight %}

Obviously a compiler lacking RTTI doesn't support `auto` or ranged-`for` either, but you get the picture. It's uglier and much more boilerplatey in practice. Now we have modern C++ implementations and I wanted to investigate doing away with this pattern, so I tried switching to `dynamic_cast`. I imagined a compiler should be able to do something smarter with a built-in language feature than with a homebrew replacement.

{% highlight cpp %}

    int count = 0;
    for (auto ptr : ptrArray) {
        if (dynamic_cast<A *>(ptr) != nullptr) {
            count++;
        }
    }
{% endhighlight %}

To test performance, I tried iterating through a 100-element array of mixed types 10,000,000 times.

* Manual type check: *1.865s*
* dynamic_cast: *16.235s*

Eek! In retrospect, it was silly to think this would be faster, because `dynamic_cast` is not really equivalent to our type check: it does much more.  It has to check not only if our object is the given type, but also if any ancestor class is.  It also has to check if our pointer is null (in my case I already know it isn't). Giving up on `dynamic_cast` I began searching for other options, and stumbled on `typeid()`.

### typeid()

`typeid()` is built into the C++ language now, but for some reason I'd never seen it used, probably because it's a symptom of bad OO design. It's surely no worse than what we're already doing, though, so let's give it a shot.

{% highlight cpp %}

    int count = 0;
    for (auto ptr : ptrArray) {
        if (typeid(*ptr) == typeid(A)) {
            count++;
        }
    }
{% endhighlight %}

* typeid() type check: *3.818s*

Much better than `dynamic_cast`, but still not very good! So what's going on here?

    testTypeId():                        # @testTypeId()
            push    rbp
            push    r14
            push    rbx
            xor     ebp, ebp
            mov     rbx, -800
            mov     r14d, typeinfo name for A
    .LBB2_1:                                # =>This Inner Loop Header: Depth=1
            mov     rax, qword ptr [rbx + ptrArray+800]
            test    rax, rax
            je      .LBB2_8
            mov     rax, qword ptr [rax]
            mov     rax, qword ptr [rax - 8]
            mov     rdi, qword ptr [rax + 8]
            cmp     rdi, r14
            je      .LBB2_5
            cmp     byte ptr [rdi], 42
            je      .LBB2_6
            mov     esi, typeinfo name for A
            call    strcmp
            test    eax, eax
            jne     .LBB2_6
    .LBB2_5:                                #   in Loop: Header=BB2_1 Depth=1
            inc     ebp
    .LBB2_6:                                #   in Loop: Header=BB2_1 Depth=1
            add     rbx, 8
            jne     .LBB2_1
            mov     eax, ebp
            pop     rbx
            pop     r14
            pop     rbp
            ret
    .LBB2_8:
            call    __cxa_bad_typeid

What's that? `call strcmp`? *`call strcmp`*? You can't be serious! And what's this `call __cxa_bad_typeid`?

That's with *libstdc++*. Let's try *libc++* instead.

* libc++ typeid() type check: *0.936s*

Wow! Heck of an improvement.

    testTypeId():                        # @testTypeId()
            push    rax
            mov     ecx, ptrArray
            xor     eax, eax
            mov     r9d, typeinfo name for A
            mov     r8d, ptrArray+800
    .LBB2_1:                                # =>This Inner Loop Header: Depth=1
            mov     rsi, qword ptr [rcx]
            test    rsi, rsi
            je      .LBB2_5
            mov     rdi, qword ptr [rcx + 8]
            test    rdi, rdi
            je      .LBB2_5
            mov     rsi, qword ptr [rsi]
            mov     rsi, qword ptr [rsi - 8]
            xor     edx, edx
            cmp     qword ptr [rsi + 8], r9
            sete    dl
            add     edx, eax
            mov     rax, qword ptr [rdi]
            mov     rsi, qword ptr [rax - 8]
            xor     eax, eax
            cmp     qword ptr [rsi + 8], r9
            sete    al
            add     eax, edx
            add     rcx, 16
            cmp     rcx, r8
            jne     .LBB2_1
            pop     rcx
            ret
    .LBB2_5:
            call    __cxa_bad_typeid


It _appears_ as though switching from libstdc++ to libc++ lets us avoid a costly string comparison.  clang/libc++ is more than four times faster than clang/libstdc++ in this case, and almost twice as fast as our homebrew RTTI.

### Null Check

There's still that awkward `call __cxa_bad_typeid`.  What's that about?  The C++ standard says: 

> When typeid is applied to a glvalue expression whose type is a polymorphic class type (10.3), the result refers to a std::type_info object representing the type of the most derived object (1.8) (that is, the dynamic type) to which the glvalue refers. If the glvalue expression is obtained by applying the unary * operator to a pointer and the pointer is a null pointer value (4.10), the typeid expression throws an exception (15.1) of a type that would match a handler of type std::bad_typeid exception (18.7.3).

Well, I already know in my context that my pointer is non-null, so this check is wasted cycles for me.  How can I convince the compiler not to check?  How about by comparing something other than a pointer?

{% highlight cpp %}

    int count = 0;
    for (auto ptr : ptrArray) {
        Base & ref = *ptr;
        if (typeid(ref) == typeid(A)) {
            count++;
        }
    }
{% endhighlight %}

* libc++ reference typeid() type check: *0.769s*

Not bad! Comparing the `typeid()` of a reference is substantially faster than comparing the typeid() of a pointer. I have no idea if there's some better idiomatic way to do this.

### What's Going On?

So what was really going on with that string comparison?  Are the libstdc++ authors just dummies?  Well, no.  Examination of both implementations suggests that they both have the same optimization, and both are switched on or off. In libstdc++:

{% highlight cpp %}

    // We can't rely on common symbols being shared between shared objects.
    bool std::type_info::
    operator== (const std::type_info& arg) const _GLIBCXX_NOEXCEPT
    {
    #if __GXX_MERGED_TYPEINFO_NAMES
      return name () == arg.name ();
    #else
      /* The name() method will strip any leading '*' prefix. Therefore
         take care to look at __name rather than name() when looking for
         the "pointer" prefix.  */
      return (&arg == this)
        || (__name[0] != '*' && (__builtin_strcmp (name (), arg.name ()) == 0));
    #endif
    }
{% endhighlight %}

...and in libc++:

{% highlight cpp %}

    #if defined(_LIBCPP_HAS_NONUNIQUE_TYPEINFO)
    // ...
        _LIBCPP_INLINE_VISIBILITY
        bool operator==(const type_info& __arg) const _NOEXCEPT
        {
          if (__type_name == __arg.__type_name)
            return true;

          if (!((__type_name & __arg.__type_name) & _LIBCPP_NONUNIQUE_RTTI_BIT))
            return false;
          return __compare_nonunique_names(__arg) == 0;
        }
    #else
    // ...
        _LIBCPP_INLINE_VISIBILITY
      bool operator==(const type_info& __arg) const _NOEXCEPT
      { return __type_name == __arg.__type_name; }
    #endif
{% endhighlight %}

In libstdc++, there's a comment elaborating on how they now turn this off by default in all circumstances because when types are dynamically loaded, they can't guarantee that all objects will have `type_info`s pointing to the same string, with some sordid history of how it used to be enabled. I don't know enough to say why libc++ doesn't seem to have this concern.

As for the libc++ version, it appears that `_LIBCPP_HAS_NONUNIQUE_TYPEINFO` is not defined on my platform, but it may be on some others.  That `_LIBCPP_NONUNIQUE_RTTI_BIT` is apparently only defined on Macs with 64-bit PowerPCs.

I tried turning on that libstdc++ flag at compile time myself:

    [dholmes@fedora test]$ clang++ -std=c++11 -O3 -D__GXX_MERGED_TYPEINFO_NAMES test.cpp -o test_typeidref_clang_merged

* __GXX_MERGED_TYPEINFO_NAMES typeid() check: *0.766s*

### A Few More Numbers

Now, all the above examples have been compiled with clang. I tested each type check method with gcc as well:

* gcc manual: *1.075s*
* gcc dynamic_cast: *16.314s*
* gcc typeid(ptr): *19.551s*
* gcc typeid(ref): *19.341s*
* gcc typeid(ref) with merged strings: *1.077s*

Notice how the non-optimized `typeid()` checks are much slower than clang, but the homebrew RTTI type check is much faster!

I also tested the `dynamic_cast` version in clang with libc++:

* clang/libstdc++ dynamic_cast: *16.235s*
* clang/libc++ dynamic_cast: *10.206s*

That's pretty noteworthy.  I think I've dug through enough assembly for tonight, though, so that exploration will have to wait for another time.

### Conclusion

So what have I learned today?

1. `typeid()`/`type_info` performance varies dramatically depending on your C++ compiler and C++ standard library implementation.
2. `typeid()` _can_ be much faster than other type checks if all the stars are aligned, or it can be extremely slow.
3. `typeid()` requires the compiler to determine if a pointer is null, so if you already know your pointer is non-null, it's more efficient to use `typeid()` on a reference instead.

Questions for another day:

1. Why is it safe for libc++ to compare pointers instead of actual strings, but not safe for libstdc++?
2. What are the causes of the performance discrepancies between gcc and clang?
3. What is the cause of the `dynamic_cast` performance discrepancy between libstdc++ and libc++?
4. What about Microsoft? Intel?
