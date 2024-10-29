---
layout: post
title: Unknowns of Ruby's Enumerable
subtitle: with examples
date: 2024-10-29 14:50 +0100
summary: |
  A collection of lesser-known methods on Enumerables.
---

The [`Enumerable` module in Ruby](https://docs.ruby-lang.org/en/master/Enumerable.html) is a collection of methods that can be used on any object that responds to the `#each` method.

```ruby
[1, 2, 3].each { |i| puts i }
```

This will print `1`, `2` and `3` on separate lines.

The `Enumerable` module also includes methods like `#map`, `#select`, `#reduce` and `#sort`.

```ruby
[1, 2, 3].map { |i| i * 2 }
```

This will return `[2, 4, 6]`.

As long as you respond to `#each`, you can use any of these methods on any object.

There is, however, a bunch of "unknown" or "forgotten", seldom used, methods that are also included in the `Enumerable` module.

### Tally

Lets start with `#tally`.
Like `#count` returns the number of elements in the collection, `#tally` will return a hash with the number of occurrences of each element in the collection.

```ruby
[1, 2, 3, 1, 2, 1].tally
```

This will return `{ 1 => 3, 2 => 2, 3 => 1 }`.

This is useful if you, for example, have a list of users each in a separate country and you want to know how many users are from each country.

```ruby
User = Data.define(:name, :country)

users = [
  User["John", "Denmark"],
  User["Jane", "Denmark"],
  User["Alice", "Sweden"],
  User["Bob", "Sweden"],
  User["Charlie", "Denmark"],
  User["Mark", "Germany"],
]

users.map(&:country).tally
```

This will return `{ "Denmark" => 3, "Sweden" => 2, "Germany" => 1 }`.

### Minmax

Mostly just forgotten, but if you need both the minimum and maximum value in a collection, you can use `#minmax`.

```ruby
[1, 2, 3, 4, 5].minmax
```

This will return `[1, 5]`.

There's also a `#minmax_by` method that returns the minimum and maximum value in the collection as determined by the given block.

### Partition

I think most are familiar with the `#group_by` method.

```ruby
User = Data.define(:name, :country)

users = [
  User["John", "Denmark"],
  User["Jane", "Denmark"],
  User["Alice", "Sweden"],
  User["Bob", "Sweden"],
  User["Charlie", "Denmark"],
  User["Mark", "Germany"],
]

users.group_by(&:country)
```

This will return a hash with the countries as keys and the users in that country as values.

`#partition`, however, takes a block and splits the collection into two groups based on the result of the block.

```ruby
[1, 2, 3, 4, 5].partition(&:even?)
```

This will return `[[2, 4], [1, 3, 5]]`.

Useful, when you know there's two different groups you want your collection split into.

### Slicing

`#slice_when` along with `#slice_before` and `#slice_after` can be seen as specialized versions of `#partition`.
Where `#partition` only splits in two, `#slice_*` will slice into as many groups as necessary.

For example:

```ruby
(1..10).slice_after { |i| i % 3 == 0 }
```

will return an enumerator with the following content:

```ruby
[[1, 2, 3], [4, 5, 6], [7, 8, 9], [10]]
```

`#slice_before` works in a similar fashion, but will include the element that caused the slice.

```ruby
(1..10).slice_before { |i| i % 3 == 0 }
# => [[1, 2], [3, 4, 5], [6, 7, 8], [9, 10]]
```

`#slice_when` yields two elements at a time to the block and will slice when the block returns `true`.

In the following example we slice when the difference between the two elements is greater than 1:

```ruby
[1, 2, 4, 9, 10, 11, 15, 16].slice_when { |i, j| i + 1 != j }
# => [[1, 2], [4], [9, 10, 11], [15, 16]]
```

### Chunk

`#chunk` is a bit more complex.
It returns an enumarator where the values are tuples, the first containing the value yielded from the block and the second containing an array of all the elements that yielded that value.

```ruby
(0..11).chunk { |i| i / 3 } # integer division
# => [[0, [0, 1, 2]], [1, [3, 4, 5]], [2, [6, 7, 8]], [3, [9, 10, 11]]]
```

`#chunk_while` is similar to `#slice_when`, but slices when the block returns `false` instead of `true`.

```ruby
[1, 2, 4, 9, 10, 11, 15, 16].chunk_while { |i, j| i + 1 != j }
# => [[1], [2, 4, 9], [10], [11, 15], [16]]
```

### Mapping

Some map methods are seldom used on `Enumerable` objects.

`#filter_map` does exactly what it says, it filters and maps.
If the block returns a truthy value, it will be included in the result.

```ruby
[1, 2, 3, 4, 5].filter_map { |i| i * 2 if i.even? }
```

This will return `[4, 8]`.

`#flat_map` (aliased as `#collect_concat`) is a combination of `#map` and `#flatten`.

Take, for example:

```ruby
[[0, 1], [2, 3]].flat_map { |e| e.map { |i| i + 100 } }
```

which is identical

```ruby
[[0, 1], [2, 3]].map { |e| e.map { |i| i + 100 } }.flatten
```

This will return `[100, 101, 102, 103]`.

### Grep

Last two methods are `#grep` and `#grep_v`.
These come directly from the Unix command line tool `grep`.

`#grep` will return an array with all the elements that match the given pattern.

```ruby
["apple", "banana", "orange"].grep(/e/)
# => ["apple", "orange"]
```

`#grep` is a shorthand for `#select` with a regex pattern:

```ruby
["apple", "banana", "orange"].select { |e| /e/ === e }
# => ["apple", "orange"]
```

`#grep_v` is named after the `-v` flag in `grep` and will return all the elements that do not match the given pattern.

```ruby
["apple", "banana", "orange"].grep_v(/e/)
# => ["banana"]
```

`#grep_v` is thus a shorthand for `#reject` with a regex pattern:

```ruby
["apple", "banana", "orange"].reject { |e| /e/ === e }
# => ["banana"]
```

---

These were just an exploration of some of the lesser-known methods in the `Enumerable` module and their uses.
