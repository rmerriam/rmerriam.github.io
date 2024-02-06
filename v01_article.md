# C++: Creating ranges::to

As I worked through the Advent of Code problems, there were a couple of times where *ranges::to* would have been helpful, except It's not available until GCC-14. I needed a break from puzzle-solving, so I worked on a version: *mys::to*.

This is my incomplete journey in creating *mys::to*.

### The Target: ranges::to

The signature for *ranges::to* is, for exposition:

```c++
 template <class ConT, class Rng, class InpT = "element type of Rng"
 auto to<ConT>(Rng<InpT>) -> ConT<InpT>;
```

The full details are at [cpprefernece](https://en.cppreference.com/w/cpp/ranges/to).

The purpose of *ranges::to* is to convert a range into a new container. It is mainly intended for invoking a pipeline and depositing the result in a container. It eliminates the need to invoke the pipeline iteratively.

Here’s an illustration:

```c++
 // get 10 elements from data and put them into vector vec
 auto pipe = data | vws::drop(10);
 auto vec = to<std::vector>(pipe);
 
```

The destination container, in this case *std::vector*, doesn’t need to specify the type of its elements. The element type is derived from *pipe* elements, a nice benefit that is a challenging requirement when writing *mys::to*.

### Simple *mys::to* Using *rng::copy*

I first worked on *mys::to<vector<char>>(<a string_view>)* by specifying the element data type. It is just a *rng::copy* inside the function.

```c++
 namespace rng = std::ranges;
 
 template<typename ConT, rng::range Rng> )
 auto to(Rng && src) -> ConT {
    ConT dst;
    rng::copy(src, std::back_inserter(dst));
    return dst; 
 }
```

It worked fine with *vector*. Then I tried all the other containers, and it didn’t work for many of them. *Std::set*, for one, cannot be copied. The [details](https://en.cppreference.com/w/cpp/algorithm/ranges/copy) for *rng::copy* show that the input and output containers must be [*indirectly_copyable*](http://en.cppreference.com/w/cpp/iterator/indirectly_copyable)*<I, O*>. Rng::copy is a *range for* using the *begin()* and *end()* iterators, pointers, to access the elements. Cpprefernce illustrates this in an [example](https://en.cppreference.com/w/cpp/algorithm/ranges/copy) implementation.

```c++
 for (; first != last; ++first, (void)++result)
   *result = *first;
 return {std::move(first), std::move(result)};
```

Many containers, like *std::set*, don’t have *std::begin()* or *std::end()*.

### A *mys::to* For *std::set*

I addressed this using *if constexr* expressions inside a *range for* to get the proper function calls to copy the elements to the new container.

```c++
 template<typename ConT, rng::range Rng>
 auto to(Rng&& src) -> ConT {
    ConT dst;
    for (auto&& s: src) {
       if constexpr (requires { dst.emplace_front(1); }) {
          dst.emplace_front(s);
       } else {
          dst.emplace(s);
       }
    }
    return dst;
 }
```

The original version of this code used many more *if constexpr* expressions, but studying the containers methods chart from [cppreference](https://en.cppreference.com/w/cpp/container) (bottom of page) reduced it to only *emplace_front* or *emplace*.

The *requires* clause determines if the destination container has the required methods. Only *forward_list* requires *emplace_front*.

I want to say this took me only minutes to reach this point, but I spent hours scratching my head while reading and re-reading *cppreference*. Some time was invested in writing a test framework to exercise each container version to ensure they all worked.

### Creating *mys::to\<container>*

The next step, and it is a big one, is to create the version that doesn’t require specifying the type of the elements.

I moved the above versions into a new namespace, *mys::detail*, and renamed them to *detail::_to* so there would be less confusion in reading them. The proper versions of *mys::to* will call these.

The new version needs to determine the type of the elements in the input range. I have seen this done but didn’t recall the details, back to the web.

The documentation for *ranges::to* on [*cppreference*](https://en.cppreference.com/w/cpp/ranges/to) reveals a version like mine but with more concepts and a second version that handles the version I want to create. Great! Some clues.

```C++
 template< template< class... > class C, ranges::input_range R >
 constexpr auto to( R&& r, Args&&... args );
```

(Note: I’m ignoring the *Args* parameter for now, at least.)

It is insufficient because it doesn’t determine the element type, i.e., *InpT* in *R<InpT>*. Looking further, both versions have all kinds of options, but it's gibberish without study. I started working through them and found the following:

```C++
 using value_type = ranges::range_value_t<R>;
```

Okay, I knew containers provide the element's value type, but this reminder was needed. Let's try it.

```C++
 template<template<class...> class ConT, rng::range Rng>
 auto to(Rng&& src) {
    return detail::_to<ConT<rng::range_value_t<Rng>>, Rng>(src);
 }
```

That's ugly, but it works. How can I specify the trailing return type without repeating all that nasty stuff?

I know there is some way to specify the type in the template parameter list. What is it? Let's look at the Ranges-v3 library for a hint. There it is:

```C++
template<template<typename> typename ConT, rng::range Rng, typename InpT = rng::range_value_t<Rng>>
auto to(Rng&& src) -> ConT <InpT> {
  return detail::_to<ConT<InpT>, Rng>(src);
}
```

Now it works. Why?

First, `template<typename> typename ConT` indicates that *ConT* is a type that takes a template parameter. In this case, it is a container with elements of type *InpT*.

Next, `typename InpT = rng::range_value_t<Rng>` retrieves the type of *Rng* and assigns it to *InpT*.

### Pipeline Failure

Does it work with input from a pipeline? Sigh, no. There's an error message about not converting an r-value to an l-value.

### Wrap Up

That’s enough for now. I have incomplete work for pipelines that's grist for a future article. This code is [available](https://gitlab.com/cpp-code-for-articles/cpp-exploration/-/tree/main/mys_to/v01?ref_type=heads) on GitLab.

This code works with all the containers that take single arguments. *Std::map* doesn’t work because it requires a *key* and a *value*. It may get a look once  pipelines are working

The journey to this point has been educational. I’ve used aspects of C++ I was aware of but never used much, like *if constexpr*.

I hope newer developers are encouraged by seeing this gray beard stumble through my effort. Folks think developers sit down, and the code pours from their fingertips. It doesn’t. Before the web, we had the three-foot rule: keep your books and documentation within a three-foot reach. Using the web means keeping *cppreference* and the search engine active all the time.

![image-20240202163730004](./header_image.png)
