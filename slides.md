<!-- .slide: data-background-image="assets/lightning.jpg" data-background-opacity="0.4" -->
## `ranges::operator|` is&nbsp;associative

<br/>
<br/>

### Jonas Greitemann
https://greitemann.dev

---

### ⚠️ Disclaimer

* This is an addendum to Tina Ulbrich's talk
* If you haven't seen her talk, go [watch it][cpponsea-talk]
* C++23 `std::ranges` assumed
* I didn't `constexpr` all the things
* No perfect forwarding

[cpponsea-talk]: https://www.youtube.com/watch?v=Ln_cVjJl680

Notes:
- MSVC has support for everything I'll show
- Godbolt links at the end
- Slideware

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
<!-- .element: class="fragment" -->

Notes:
- Tina used range-v3's `sliding` view
- **Step**: Drop-in replacement
- **Step**: {fmt} library
- Eager

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

Notes:
- `ranges::to` dropped
- **Step**: Returns lazy view; constrained
- work happens in `fmt::print`
- no heap allocation

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

Notes:
- So far: `span` (contiguous + sized)
- Doesn't work with e.g. `forward_range`s

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

Notes:
- Becomes a function template
- `slide` requires at least a `forward_range`
- **Step**: Doesn't compile with `mean` as-is

----

<!-- .slide: data-auto-animate data-auto-animate-restart -->
#### Generalizing `mean` to work with ranges
```cpp
constexpr auto mean(std::span<double const> rng) -> double {
  return std::reduce(rng.begin(), rng.end(), 0.) / rng.size();
}
```
<!-- .element: class="mark error squiggle-params" data-id="mean" -->

Notes:
- Also accepts `span`
- Worked before b/c `slide` preserves contiguity
- Needs to be generalized, too

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

Notes:
- Also becomes a function template
- `slide` yields `sized_range`s
- Problem:
  - `<numeric>` algorithms not yet rangified
  - Ranges iterators can have sentinels

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

Notes:
- `common_view` works around this problem
- This version compiles perfectly well
- **Step**: But template cannot be used a function ref

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

Notes:
- Turn `mean` into a function object (e.g. lambda)
- `mean` itself is not a template
- Call operator is generic

---

<!-- .slide: data-auto-animate data-auto-animate-restart -->
#### Functionally complete, but ugly...
```cpp
fmt::print("{}\n", sliding_mean(std::views::iota(0, 30)
                                | std::views::filter(is_even))
                   | std::views::transform(round_to_int));
```
<!-- .element: class="mark no-error" data-id="pipe-usage" -->

Notes:
- Functionally where we want to be (lazy)
- Awkward to write
- Potentially confusing (eager → lazy)

----

<!-- .slide: data-auto-animate -->
#### Can we use pipeline syntax for user-defined range adaptors?
```cpp
fmt::print("{}\n", std::views::iota(0, 30)
                   | std::views::filter(is_even)
                   | sliding_mean
                   | std::views::transform(round_to_int));
```
<!-- .element: class="mark question" data-id="pipe-usage" -->

Notes:
- Satisfying
- Avoids confusion: laziness implied
- Difficult to enable in C++20 w/o injecting in `std::ranges`

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
                   | std::views::filter(is_even)
                   | sliding_mean
                   | std::views::transform(round_to_int));
```
<!-- .element: class="mark no-error" data-id="pipe-usage" -->

[p2387]: https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2021/p2387r3.html

Notes:
- Paper by Barry Revzin; accepted for C++23
- CRTP base class `range_adaptor_closure`
- Our function becomes its call operator
- Customization point; not too much boilerplate

----

<!-- .slide: data-auto-animate data-auto-animate-restart -->
#### `std::ranges::operator|` is associative

Notes:
- There's a much simpler way for composite adaptors
- Circle back to title

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
                   | std::views::filter(is_even)
                   | sliding_mean
                   | std::views::transform(round_to_int));
```
<!-- .element: class="mark no-error" data-id="pipe-usage" -->

Notes:
- Specify function object directly by composition
- "Point-free" programming
- Constraints of `slide` automatically carry over
- Works in C++20 w/o support for P2387

---

![assets/qr.svg](assets/qr.svg)
<!-- .element: class="qr" -->

### That's it

* Blog post, Godbolt links & slides: [https://greitemann.dev/pipe-assoc][1]
* <!-- .element: class="no-bullets" --> Find me online
  * [@jgreitemann](https://github.com/jgreitemann)
  * [@jg424](https://twitter.com/jg424)
  * [@jgreitemann@fosstodon.org](https://fosstodon.org/@jgreitemann)
* Questions?

[1]: https://greitemann.dev/pipe-assoc

Notes:
- Questions?
- Enjoy the main talk
