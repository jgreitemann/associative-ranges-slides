<!-- .slide: data-background-image="lightning.jpg" data-background-opacity="0.4" -->
## `ranges::operator|` is&nbsp;associative
<!-- .element: class="shadow" -->

<br/>
<br/>

### Jonas Greitemann
<!-- .element: class="shadow" -->
https://greitemann.dev
<!-- .element: class="shadow" -->

---

### ⚠️ Disclaimer

* This is an addendum to Tina Ulbrich's talk
* If you haven't seen her talk, go [watch it][cpponsea-talk]
* C++23 `std::ranges` assumed
* I didn't `constexpr` all the things
* No perfect forwarding

[cpponsea-talk]: https://www.youtube.com/watch?v=Ln_cVjJl680

---

<!-- .slide: data-auto-animate -->
#### Tina's `sliding_mean` example

```cpp[]
auto sliding_mean(std::span<double const> rng)
    -> std::vector<double>
{
  auto out = std::vector<double>(rng.size() - 4);
  for (size_t i = 2; i < rng.size() - 2; ++i) {
      out[i - 2] = mean(std::array{
        rng[i-2], rng[i-1], rng[i], rng[i+1], rng[i+2]
      });
  }
  return out;
}
```
<!-- .element: data-id="tinas-example" -->

----

<!-- .slide: data-auto-animate -->
#### Tina's `sliding_mean` example

```cpp [|2,6]
auto sliding_mean(std::span<double const> rng)
    -> std::vector<double>
{
  return rng | std::views::slide(5)
             | std::views::transform(mean)
             | std::ranges::to<std::vector>();
}
```
<!-- .element: data-id="tinas-example" -->

```cpp
std::array a = {3., 1., 4., 1., 5., 9., 2., 6., 5., 3.};
fmt::print("{}\n", sliding_mean(a)
                   | std::views::transform(round_to_int));
```

---

#### Eager evaluation
<!-- .element: data-id="gif-heading" -->

<!-- .slide: data-auto-animate data-auto-animate-restart -->
<iframe data-src="https://giphy.com/embed/ATXUOl4iE5ZPq" width="383" height="480" frameBorder="0" class="giphy-embed" allowFullScreen></iframe>

----

#### Lazy evaluation
<!-- .element: data-id="gif-heading" -->

<!-- .slide: data-auto-animate -->
<iframe data-src="https://giphy.com/embed/WoCxkkpiweO6Q" width="480" height="284" frameBorder="0" class="giphy-embed" allowFullScreen></iframe>

----

<!-- .slide: data-auto-animate data-auto-animate-restart -->
#### Returning lazy views instead
```cpp [|2]
auto sliding_mean(std::span<double const> rng)
    -> std::ranges::random_access_range auto
{
  return rng | std::views::slide(5)
             | std::views::transform(mean);
}
```
<!-- .element: data-id="gen-out" -->

```cpp
std::array a = {3., 1., 4., 1., 5., 9., 2., 6., 5., 3.};
fmt::print("{}\n", sliding_mean(a)
                   | std::views::transform(round_to_int));
```

---

<!-- .slide: data-auto-animate -->
#### Can we be even more lazy?
```cpp []
auto sliding_mean(std::span<double const> rng)
    -> std::ranges::random_access_range auto
{
  return rng | std::views::slide(5)
             | std::views::transform(mean);
}
```
<!-- .element: class="mark error squiggle-params" data-id="gen-out" -->

```cpp
fmt::print("{}\n", sliding_mean(std::views::iota(0, 30)
                                | std::views::filter(is_even))
                   | std::views::transform(round_to_int));
```
<!-- .element: class="mark question" -->

----

<!-- .slide: data-auto-animate -->
#### Accepting ranges as function arguments
```cpp [1,2|1,2,5]
auto sliding_mean(std::ranges::forward_range auto rng)
    -> std::ranges::forward_range auto
{
  return rng | std::views::slide(5)
             | std::views::transform(mean);
}
```
<!-- .element: class="mark error squiggle-mean" data-id="gen-out" -->

```cpp
fmt::print("{}\n", sliding_mean(std::views::iota(0, 30)
                                | std::views::filter(is_even))
                   | std::views::transform(round_to_int));
```

----

<!-- .slide: data-auto-animate data-auto-animate-restart -->
#### Generalizing `mean` to work with ranges
```cpp
constexpr auto mean(std::span<double const> rng) -> double {
  return std::reduce(rng.begin(), rng.end(), 0.) / rng.size();
}
```
<!-- .element: class="mark error squiggle-params" data-id="mean" -->

----

<!-- .slide: data-auto-animate -->
#### Generalizing `mean` to work with ranges
```cpp
constexpr auto mean(std::ranges::sized_range auto rng)
    -> double
{
  using std::ranges::begin;
  using std::ranges::end;
  using std::ranges::size;
  return std::reduce(begin(rng), end(rng), 0.) / size(rng);
}
```
<!-- .element: class="mark error squiggle-return" data-id="mean" -->

----

<!-- .slide: data-auto-animate -->
#### Generalizing `mean` to work with ranges
```cpp
constexpr auto mean(std::ranges::sized_range auto rng)
    -> double
{
  using std::ranges::begin;
  using std::ranges::end;
  using std::ranges::size;
  auto c = rng | std::views::common;
  return std::reduce(begin(c), end(c), 0.) / size(rng);
}
```
<!-- .element: class="mark no-error" data-id="mean" -->

```cpp
   ... | std::views::transform(mean)
```
<!-- .element: class="fragment mark error" -->

----

<!-- .slide: data-auto-animate -->
#### Generalizing `mean` to work with ranges
```cpp
inline constexpr auto mean =
    [](std::ranges::sized_range auto rng) -> double {
      using std::ranges::begin;
      using std::ranges::end;
      using std::ranges::size;
      auto c = rng | std::views::common;
      return std::reduce(begin(c), end(c), 0.) / size(rng);
    };
```
<!-- .element: class="mark no-error" data-id="mean-functor" -->

```cpp
   ... | std::views::transform(mean)
```
<!-- .element: class="mark no-error" -->

---

<!-- .slide: data-auto-animate data-auto-animate-restart -->
#### Functionally complete, but ugly...
```cpp
fmt::print("{}\n", sliding_mean(std::views::iota(0, 30)
                                | std::views::filter(is_even))
                   | std::views::transform(round_to_int));
```
<!-- .element: class="mark no-error" data-id="pipe-usage" -->

----

<!-- .slide: data-auto-animate -->
#### Can we use pipeline syntax for user-defined range adaptors?
```cpp
fmt::print("{}\n", std::views::iota(0, 30)
                   | std::views::filter(is_even))
                   | sliding_mean
                   | std::views::transform(round_to_int));
```
<!-- .element: class="mark question" data-id="pipe-usage" -->

----

<!-- .slide: data-auto-animate -->
#### [P2387: Pipe support for user-defined range adaptors][p2387]
```cpp
struct sliding_mean_fn
  : std::ranges::range_adaptor_closure<sliding_mean_fn>
{
  constexpr auto operator()(forward_range auto&& rng) const
      -> std::ranges::forward_range auto
  {
    return rng | std::views::slide(5)
               | std::views::transform(mean);
  }
};

inline constexpr sliding_mean_fn sliding_mean;
```

```cpp
fmt::print("{}\n", std::views::iota(0, 30)
                   | std::views::filter(is_even))
                   | sliding_mean
                   | std::views::transform(round_to_int));
```
<!-- .element: class="mark no-error" data-id="pipe-usage" -->

[p2387]: https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2021/p2387r3.html

----

<!-- .slide: data-auto-animate data-auto-animate-restart -->
#### `std::ranges::operator|` is associative

----

<!-- .slide: data-auto-animate -->
#### `std::ranges::operator|` is associative

```cpp
(rng | slide(5)) | transform(mean)
```
<!-- .element: class="code-centered" -->

⇕

```cpp
rng | (slide(5) | transform(mean))
```
<!-- .element: class="code-centered" -->

----

<!-- .slide: data-auto-animate -->
#### `std::ranges::operator|` is associative

```cpp
inline constexpr auto sliding_mean =
    std::views::slide(5) | std::views::transform(mean);
```

```cpp
fmt::print("{}\n", std::views::iota(0, 30)
                   | std::views::filter(is_even))
                   | sliding_mean
                   | std::views::transform(round_to_int));
```
<!-- .element: class="mark no-error" data-id="pipe-usage" -->

---

![qr.svg](qr.svg)
<!-- .element: class="qr" -->

### That's it

* Blog post, Godbolt links & slides: [https://greitemann.dev/pipe-assoc][1]
* Questions?
* Enjoy the main talk
<!-- .element: class="fragment" -->

[1]: https://greitemann.dev/pipe-assoc
