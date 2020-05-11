+++
title = "emplace_back vs push_back"
date = 2020-05-11T13:00:26+02:00
draft = false

# Tags and categories
# For example, use `tags = []` for no tags, or the form `tags = ["A Tag", "Another Tag"]` for one or more tags.
tags = []
categories = []

# Featured image
# Place your image in the `static/img/` folder and reference its filename below, e.g. `image = "example.jpg"`.
# Use `caption` to display an image caption.
#   Markdown linking is allowed, e.g. `caption = "[Image credit](http://example.org)"`.
# Set `preview` to `false` to disable the thumbnail in listings.
[header]
image = ""
caption = ""
preview = true
+++

# `emplace_back` vs `push_back`

I have now ran into the discussion of whether one should use `emplace_back` instead of `push_back` in C++ several times and I wanted to document my current view on it.

[`emplace_back`](http://www.cplusplus.com/reference/vector/vector/emplace_back/) came with C++11, I'm not sure when it was introduced exactly. Both of these methods, (and `insert` and `emplace`), are ways to insert data into standard library containers. `emplace_back` is for the dynamic array `std::vector`. There is a very subtle difference between the two:

1. `push_back` calls the constructor of the data that is being pushed, and then moves it to the container.
2. `emplace_back` "constructs in place", so one skips an extra move operation, potentially creating faster bytecode.

If you are new to C++, then this looks like an obvious choice. Pick the one that is faster, end of story, why should anyone even be using `push_back`... If you google this [`emplace_back` vs `push_back`](https://stackoverflow.com/questions/4303513/push-back-vs-emplace-back), then you are taken to an old stackoverflow question around the time this was introduced, where the conclusion is also that it is kind of obvious that you should use this more efficient way of inserting data into your container.

## Be skeptical

After googling a bit more I found [this post](https://abseil.io/tips/112). Which kind of tells you to be careful! The google c++ styleguide does not explicitly state which of the two you should use. But if you read their bit on [implicit conversion](https://google.github.io/styleguide/cppguide.html#Implicit_Conversions), it is clear that this is not completely obvious. The following code should make it clear why this is risky business:

```{cpp}
#include<vector>                                                                                                                                                                                                   
#include<iostream>

int main(){
  // Basic example
  std::vector<int> data;
  data.push_back(10);
  data.emplace_back(20);

  // More tricky example
  std::vector<std::vector<int>> data_vec;
  //data_vec.push_back(10); // Throws error!!!!
  data_vec.emplace_back(20); // Compiles with no issue
  std::cout << "data_vec size: " << data_vec.size() << std::endl;
  std::cout << "data_vec[0] size: " << data_vec[0].size() << std::endl;
  return 0;
}
```

So if I compile this with the commented out line **not commented out** I get:

```
$ g++ test.cpp -o test
test.cpp: In function ‘int main()’:
test.cpp:11:24: error: no matching function for call to ‘std::vector<std::vector<int> >::push_back(int)’
   data_vec.push_back(10);
                        ^
... Some more verbose diagnostic
```

So we get an error. In the extra diagnostic we get `no known conversion for argument 1 from ‘int’ to ‘const value_type& {aka const std::vector<int>&}’`. So there is really no conversion here that makes sense.

What is alarming is that if we comment out the trouble line, this works fine and when we run this we get:

```
$ ./test
data_vec size: 1
data_vec[0] size: 20
```

So what was the difference? For `emplace_back` we **forward the arguments to the constructor**, adding a new `std::vector<int> new_vector_to_add(20)` to `data_vec`.

## So what is the issue?

The problem is that this was not caught at compile time, and if this was not the intended behavior, we have caused a runtime error, which are generally harder to deal with. So let's catch this somehow! So you might think that you can just add some warning flags, e.g. `-Wall`. But this program compiles fine with `-Wall`. `-Wall` contains narrowing, but it does not contain conversion for some reason, so we can add `-Wconversion`, but it still compiles fine without any warnings!

```
$ g++ -Wall -Wconversion test.cpp -o test
```

The problem is that this conversion is happening in a system header, so we also need `-Wsystem-headers` to catch this:

```
$ g++ -Wconversion -Wsystem-headers test.cpp -o test
In file included from /usr/include/c++/7/vector:60:0,
                 from test.cpp:1:
/usr/include/c++/7/bits/stl_algobase.h: In function ‘constexpr int std::__lg(int)’:
/usr/include/c++/7/bits/stl_algobase.h:1001:44: warning: conversion to ‘int’ from ‘long unsigned int’ may alter its value [-Wconversion]
   { return sizeof(int) * __CHAR_BIT__  - 1 - __builtin_clz(__n); }
                                            ^
/usr/include/c++/7/bits/stl_algobase.h: In function ‘constexpr unsigned int std::__lg(unsigned int)’:
/usr/include/c++/7/bits/stl_algobase.h:1005:44: warning: conversion to ‘unsigned int’ from ‘long unsigned int’ may alter its value [-Wconversion]
   { return sizeof(int) * __CHAR_BIT__  - 1 - __builtin_clz(__n); }
                                            ^
In file included from /usr/include/c++/7/bits/stl_algobase.h:63:0,
                 from /usr/include/c++/7/vector:60,
                 from test.cpp:1:
/usr/include/c++/7/ext/numeric_traits.h: In instantiation of ‘const short int __gnu_cxx::__numeric_traits_integer<short int>::__min’:
/usr/include/c++/7/bits/istream.tcc:138:54:   required from here
/usr/include/c++/7/ext/numeric_traits.h:58:35: warning: conversion to ‘short int’ alters ‘int’ constant value [-Wconversion]
       static const _Value __min = __glibcxx_min(_Value);
                                   ^~~~~~~~~~~~~
In file included from /usr/include/c++/7/bits/basic_string.h:6361:0,
                 from /usr/include/c++/7/string:52,
                 from /usr/include/c++/7/bits/locale_classes.h:40,
                 from /usr/include/c++/7/bits/ios_base.h:41,
                 from /usr/include/c++/7/ios:42,
                 from /usr/include/c++/7/ostream:38,
                 from /usr/include/c++/7/iostream:39,
                 from test.cpp:2:
/usr/include/c++/7/ext/string_conversions.h: In instantiation of ‘_Ret __gnu_cxx::__stoa(_TRet (*)(const _CharT*, _CharT**, _Base ...), const char*, const _CharT*, std::size_t*, _Base ...) [with _TRet = long int; _Ret = int; _CharT = char; _Base = {int}; std::size_t = long unsigned int]’:
/usr/include/c++/7/bits/basic_string.h:6373:19:   required from here
/usr/include/c++/7/ext/string_conversions.h:88:8: warning: conversion to ‘int’ from ‘long int’ may alter its value [-Wconversion]
  __ret = __tmp;
        ^
/usr/include/c++/7/ext/string_conversions.h: In instantiation of ‘_Ret __gnu_cxx::__stoa(_TRet (*)(const _CharT*, _CharT**, _Base ...), const char*, const _CharT*, std::size_t*, _Base ...) [with _TRet = long int; _Ret = int; _CharT = wchar_t; _Base = {int}; std::size_t = long unsigned int]’:
/usr/include/c++/7/bits/basic_string.h:6479:19:   required from here
/usr/include/c++/7/ext/string_conversions.h:88:8: warning: conversion to ‘int’ from ‘long int’ may alter its value [-Wconversion]

```

And boom.... look at all that verbose output. Skimming through a sea of warnings, it would be rather tricky to nail this down.

## `emplace_back` is a premature optimization.

Going from `push_back` to `emplace_back` is a small change that can wait. For safety, reliability and maintainability reasons, it is better to write the code with `push_back`. This reduces the chance of pushing an unwanted hard to find implicit conversion into the codebase. When one can finally get to profiling the code, this is an opportunity to replace some of those `push_back` with `emplace_back` if there is any significant gain.
