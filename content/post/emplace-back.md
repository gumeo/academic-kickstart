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

tl;dr Choose `push_back` and only later change to `emplace_back` if it provides significant speed improvements when profiling.

I have repeatedly run into the choice of using `emplace_back` instead of `push_back` in C++. This short blog post serves as my take on this decision.

Both of the methods in the title, along with `insert` and `emplace`, are ways to insert data into standard library containers. `emplace_back` is for adding a single element to the dynamic array `std::vector`. There is a somewhat subtle difference between the two:

1. `push_back calls` the constructor of the data that you intend to push and then pushes it to the container.
2. `emplace_back` "constructs in place", so one skips an extra move operation, potentially creating faster bytecode.

One could naively choose the faster method. Why even offer a slower alternative? Searching for the problem online yields a lengthy discussion on the [issue (emplace_back vs push_back)](https://stackoverflow.com/questions/4303513/push-back-vs-emplace-back). In summary, the discussion leans towards choosing the more efficient `emplace_back` to insert data into your container, however the reason it is not completely clear.

## Be careful

After searching a bit more I found [this post](https://abseil.io/tips/112), which stresses how careful one should be with this decision. Which kind of tells you to be careful! To further stress the ambiguity of the matter, the google c++ style guide does not provide an explicit preference. However, in their section on [implicit conversion](https://google.github.io/styleguide/cppguide.html#Implicit_Conversions), it becomes clear that the decision between the two methods is not completely obvious. The following code should make it clear why `emplace_back` is not worth the risk: 


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

Uncommenting the line `data_vec.push_back(10)` yields the following compilation error.

```
$ g++ test.cpp -o test
test.cpp: In function ‘int main()’:
test.cpp:11:24: error: no matching function for call to ‘std::vector<std::vector<int> >::push_back(int)’
   data_vec.push_back(10);
                        ^
... Some more verbose diagnostic
```

So we get an error. An extra diagnostic provides us with the following: `no known conversion for argument 1 from ‘int’ to ‘const value_type& {aka const std::vector<int>&}’`. So it seems there is no conversion here that makes sense.

The compilation completes without errors if we comment out the trouble line. Running the code yields the following result:


```
$ ./test
data_vec size: 1
data_vec[0] size: 20
```

What is the difference? For `emplace_back` we **forward the arguments to the constructor**, adding a new `std::vector<int> new_vector_to_add(20)` to `data_vec`.

## Why is this a problem?

The problem is that we are unaware of the problem at compile-time. If this was not the intended behavior, we have caused a runtime error, which is generally harder to fix. Let us catch the issue somehow. You might wonder that some warning flags, e.g., `-Wall` could reveal the issue. However, the program compiles fine with `-Wall`. `-Wall` contains narrowing, but it does not contain conversion. Further, adding `-Wconversion` yields no warnings!

```
$ g++ -Wall -Wconversion test.cpp -o test
```

The problem is that this conversion is happening in a system header, so we also need `-Wsystem-headers` to catch the issue.

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

And there you have it - look at all the verbose output. The problem is not apparent from this wall of text for the uninitiated.

## `emplace_back` is a premature optimization.

Going from `push_back` to `emplace_back` is a small change that can usually wait. For safety, reliability, and maintainability reasons, it is better to write the code with `push_back`. This choice reduces the chance of pushing an unwanted hard to find implicit conversion into the codebase. Profiling the code might reveal opportunities to replace some `push_back` calls with `emplace_back`, but remember when optimizing to tread carefully.
