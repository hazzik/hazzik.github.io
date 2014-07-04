---
layout: post
title: "Learn How to Use IDcitionary<,>.TryGetValue!"
date: 2014-07-04T14:36:35+12:00
tags:
- Best practicies
- .NET
---
I'm going nuts every time I see the code similar to the following snippet.

```csharp
/* Given:
 * IDictionary<TKey, TValue> dictionary;
 * TKey key;
 */
if (dictionary.ContainsKey(key) 
{
    var value = dictionary[key];
    // do something with retrieved value
}

```
You should really use [TryGetValue](http://msdn.microsoft.com/en-us/library/bb347013) instead:

```csharp
/* Given:
 * IDictionary<TKey, TValue> dictionary;
 * TKey key;
 */
TValue value;
if (dictionary.TryGetValue(key, out value) 
{
    // do something with retrieved value
}
```

And here is a couple of reasons why:

* It is more performant as the location of the value calculated only once. And in fact implementation of `ContainsKey`, `Item` and `TryGetValue` is exactly the same.
* In the first example the dictionary could be changed between `ContainsKey` and `Item` method calls, and the retrieving of the item could fail in multi-threaded environment.

I was so crazy about this, so couple of month ago I wrote [a plugin](https://resharper-plugins.jetbrains.com/packages/ReSharper.DictionaryHelper/) for [ReSharper](http://jetbrains.com/resharper) which can analyze and optimize such subsequent calls.