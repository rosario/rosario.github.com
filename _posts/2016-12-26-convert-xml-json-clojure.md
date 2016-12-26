---
layout: post
title: "Convert XML to JSON in Clojure"
categories: [clojure, xml, json, recursion]
---

Mainly as an exercise in recursion and because I needed something quick I decided to implement
my own XML/JSON converter. Here follow the code with the explanation.


### Parse a XML string

The standard `clojure.xml/parse` is excellent for parsing,  as
explained in [xml for fun and profit](http://blog.korny.info/2014/03/08/xml-for-fun-and-profit.html)
it can be done with this simple one-liner:


```clojure
    (defn parse [s] (clojure.xml/parse (java.io.ByteArrayInputStream. (.getBytes s))))
```

However `clojure.xml/parse` returns a clojure hash with the following structure:

* _:attrs_ contains the attributes
* _:content_ contains the nested structure of the XML
* _:tag_ is the name of the XML tag

For example, if we have the following XML:

```HTML
    <hello>bravo</hello>
```

The result of calling `(parse "<hello>bravo</hello>")` would be:


```clojure
    {:attrs nil, :content ["bravo"], :tag :hello}
```

Instead I wanted to produce the following output:

```clojure
    {:hello "bravo"}
```

After that, it's very easy to convert the previous Clojure map in JSON using
[cheshire](https://github.com/dakrone/cheshire)



### The base case

When writing a recursive function, it's good practise to start with the base case: ie. _when the function
is defined for well known cases_. If a recursive function operates on numbers
the typical base case would be a... yes, just a number. If the function operates on lists, the base case
is typically the empty list. Otherwise if the function return hashes (or maps
or Objects if you program in Javascript) the common base case is the empty object.

We'll define function called `xml->json` and consider the following base cases:

```clojure
    (defn xml->json [element]
      (cond
        (nil? element) nil
        (string? element) element
        (and (map? element) (empty? element)) {}
        :else nil))
```


### Recursion

The simplest XML to convert is `<hello>bravo</hello>`.

```clojure
    (def simple-input (parse "<hello>bravo</hello>"))
    ;{:attrs nil, :content ["bravo"], :tag :hello}
```

Let's add recursive condition to support that input


```clojure
    (defn xml->json [element]
      (cond
        (nil? element) nil
        (string? element) element
        (and (map? element) (empty? element)) {}
        (map? element) {(:tag element) (xml->json  (:content element))}
        :else nil))
```

The previous code says: If the element is a map then it returns a new map with key equals to `(:tag element)`
and the value equals to `xml->json` applied to the `(:content element)`.

If we try to run the previous code, we will get:

```clojure
    (xml->json simple-input)
    ;{:hello nil}
```

We get back a `nil` value because `(:content element)` is a list of _elements_, and we don't have
a case defined for that (the code reach the `:else` condition). Let's fix this problem, adding
a condition when there is a list of elements:


```clojure
    (defn xml->json [element]
      (cond
        (nil? element) nil
        (string? element) element
        (and (map? element) (empty? element)) {}
        (sequential? element) (map xml->json element)
        (map? element) {(:tag element) (xml->json  (:content element))}
        :else nil))
```

The previous code `(sequential? element) (map (partial xml->json) element)` means:

> If the element is a sequence, we want to apply `xml->json` to every items.

If we try again to run the modified version, we will see that:

```clojure
    (xml->json simple-input)
    ; {:hello ("bravo")}
```

That's better, we eventually get the value `"bravo"`, but it is not _right_ yet.
In JSON we want to get this:

```JSON
    {"hello": "bravo"}
```

The previous code instead returns the following JSON:

```JSON
    {"hello": ["bravo"]}
```

>Note: We actually get a clojure map that would be converted into JSON using Cheshire.


It seems that the code (at it stands) works well for those element having a list of nested sub-elements.
Just for testing purposes, we can try to run the following:


```clojure
    (def simple-nested (parse "<div><h1>hello</h1><h1>ciao</h1></div>"))
    (xml->json simple-nested)
    ; {:div ({:h1 ("hello")} {:h1 ("ciao")})}
```

The _map_ is actually working, but we still have a problem there. If the sequence has only one element
we do not want to return the _list_, instead we want to return the element itself.

Let's fix the code again, considering that:

```clojure
    (defn xml->json [element]
      (cond
        (nil? element) nil
        (string? element) element
        (and (map? element) (empty? element)) {}
        (sequential? element) (if (> (count element) 1)
                        (map xml->json element)
                        (xml->json  (first element)))
        (map? element) {(:tag element) (xml->json  (:content element))}
        :else nil))
```

The previous code does the following:

> Assuming the element is a sequence, if there are more than one items then
we apply the `map` to every items, otherwise we call `xml->json` on the first item.

If we try now to run the previous code, it will produce the following output:

```clojure
    (xml->json simple-nested)
    ; {:div ({:h1 "hello"} {:h1 "ciao"})}
```


### Special case: Array or Object?
There are few more special cases to consider. For example the following XML:

```HTML
    <div>
        <h1> hello </h1>
        <h1> ciao </h1>
    </div>
```

would be converted into this JSON

```JSON
    {
        "div": [
            {"h1": "hello"},
            {"h1": "ciao"}
        ]
    }
```

It's an Array of Objects! Instead, this XML:
```XML
    <person>
        <fullname> Salvatore </fullname>
        <address> Catania </address>
    </person>
```

It's *better* represented by the following JSON:

```JSON
    {
        "person": {
            "fullname": "Salvatore",
            "address": "Catania"
        }
    }
```

Instead, the function `xml->json` would produce:

```JSON
    {
        "person": [
            {"fullname": "Salvatore"},
            {"address": "Catania"}
        ]
    }
```

Array of Objects! Although both XML contains a sequence of inner elements, the JSON should be different.
Why?

It's mostly personal taste, both JSON representations are valid, but I think that's easier to
have an Object instead of an Array. For example, it's easier to access the property `fullname` like this:

```javascript
    // Javascript code ahead!
    var data = {
        "person": {
            "fullname": "Salvatore",
            "address": "Catania"
        }
    }
    console.log(data.person.fullname); // We can easily access the fullname property
```

If we would have instead an array, it won't be possible to use the _dot notation_.

Having explained that, I'll make the following _arbitrary decision_:

> If the list of elements contains the **same tag** (a list of `<h1>` for example) then it's fine
to produce JSON Array. Otherwise, if the list of elements contains **different
tags** (like in the previous example, we had `<fullname>` and `<address>`) then we produce
an JSON Object.


Let's defined a helper function that checks if a sequence of elements are different:

```clojure
    (defn different-keys? [content]
      (when content
        (let [dkeys (count (filter identity (distinct (map :tag content))))
              n (count content)]
          (= dkeys n))))
```

Now, we can use the `different-keys?` in our recursive `xml->json`:

```clojure
    (defn xml->json [element]
      (cond
        (nil? element) nil
        (string? element) element
        (and (map? element) (empty? element)) {}
        (sequential? element) (if (> (count element) 1)
                                  (if (different-keys? element)
                                    (reduce into {} (map (partial xml->json ) element))
                                    (map xml->json element))
                                  (xml->json  (first element)))
        (map? element) {(:tag element) (xml->json  (:content element))}
        :else nil))
```

The following line `(reduce into {} (map (partial xml->json ) element))` means:

> Apply `xml->json` to every items, and reduce the list into a hash. In other words, the function `map` returns a list that's why we convert it back into a hash.

Finally we have a `xml->json` that works for most the cases, and parses into Array or Object
depending on the tags.

### Are we there yet?

Not yet! There's one more thing: XML _attributes_!
We want the ability to support attributes, for example the following XML:


```XML
    <person gender="female">
        <name> Carmela </name>
        <address> Siracusa </address>
    </person>
```

The attribute `gender` needs to be converted into a meaningful JSON object. Again, the following
decision is arbitrary, and surely can be implemented in other ways. I wanted to group _attributes_ together
inside one JSON keyword (other implementations might want to use the "@" chars for example):

The final code is:

```clojure
(defn xml->json [element]
  (cond
    (nil? element) nil
    (string? element) element
    (sequential? element) (if (> (count element) 1)
                           (if (different-keys? element)
                             (reduce into {} (map (partial xml->json ) element))
                             (map xml->json element))
                           (xml->json  (first element)))
    (and (map? element) (empty? element)) {}
    (map? element) (if (:attrs element)
                    {(:tag element) (xml->json (:content element))
                     (keyword (str (name (:tag element)) "Attrs")) (:attrs element)}
                    {(:tag element) (xml->json  (:content element))})
    :else nil))
```

This line of code `(keyword (str (name (:tag element)) "Attrs")) (:attrs element)` means:

> Convert the value of `(:tag element)` into another keyword with a postfixed "Attrs", the `(:attrs element)` will follow.


## Conclusion

XML to JSON is a very common procedure. Clojure core support xml via `clojure.xml`, and the
de-facto method to query/access and manipulate XML is using zippers.

I needed a simpler way to just transform XML into JSON, and came up with a recursive and simple solution.
I don't think it's optimised to handle large XML, but it shouldn't be that bad either.

Feel free to use the code: [https://github.com/rosario/json-parse](https://github.com/rosario/json-parse).

















