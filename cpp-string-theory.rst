:Author:
  Dean Michael Berris <mikhailberis@gmail.com>
:Creation Date:
  Jan. 31, 2011
:Version:
  Feb. 1, 2011 -- Updates and more details

C++ String Theory
=================

Abstract
--------

C++ is a wonderfully powerful programming language but it does not have a native
string type. This is a good and bad thing at the same time. To solve that issue
the ISO C++ Standard Committee defines a library implementation of a string data
type. Unfortunately that didn't stop anybody from writing their own string
implementation. This paper attempts to present a theory on how the string data
type should be implemented in the context of C++ and how it can help C++
programmers in general deal with a better string abstraction in a larger context
of applications. [#]_

.. [#] The title of the paper is a play on the unified theory for everything in
   Physics; although this is not a unified theory for everything in C++, this is
   a unified theory on strings and how they should be dealt with in C++.

Introduction
------------

This paper was inspired by the discussions that have transpired in the Boost C++
Developers mailing list where some issues with regard to text processing and
string encoding came up. There were lots of ideas thrown around regarding how
strings should be dealt with and how the standard string (``std::string`` from here
on out) could be made to by default be encoded in UTF-8 [#]_. One thing that
wasn't being discussed is why ``std::string`` isn't a suitable string data structure
implementation. [#]_

.. [#] http://en.wikipedia.org/wiki/UTF-8
.. [#] http://thread.gmane.org/gmane.comp.lib.boost.devel/213235

I then thought it was a good time to point out that there are a lot of problems
with ``std::string`` and that one of the big problems with it is that the string it
represents is mutable. This is then where things started to get into the point
of defining what a string should be, what a string is as how others see it, and
that whether a more appropriate name for the concept that I was presenting is
not *string*. This is around the time I decided that I needed to write this
stuff up into a document that others can pass around and comment on whole-sale
rather than hash things out in a public mailing list.

This document is thus a result of days worth of thinking about the matter and
whether or not the way us C++ programmers (and other programmers with languages
allowing for string mutability) think about strings is wrong. As a matter of
opinion, I think it's time that there be something called *string theory* for
C++.

Problem Statement
-----------------

To put it bluntly, this is the problem I'm trying to address:

    ``std::string`` is broken.

.. note:: This is the part of the paper where I explain why I think the
   ``std::string`` type as it's defined and as it's implemented and specified is
   broken. The complete statement of a problem is not as succinct as above, but
   the idea is the same: *The ``std::string`` interface and usage semantics
   encourages bad implementations and habits that can otherwise be avoided by
   a more thorough and thought-through definition of string type.*

I think there will be people who will lash out at that statement, maybe see it
as heresy or blasphemy. There are some people other than me who seem to think so
and came up with different implementations of strings. I won't list them all
down, but people at SGI implemented a fascinating data structure called
``rope``. [#]_ That implementation doesn't seem to get the same attention that
``std::string`` does mostly because it's not part of the C++ standard library.

.. [#] http://www.cs.ubc.ca/local/reading/proceedings/spe91-95/spe/vol25/issue12/spe986.pdf

One thing that the C++ implementation of the ``rope`` data type that I
personally think should not have been introduced is the notion of mutability.
Because you can still modify characters inside a ``rope`` the operation is
consequently very expensive. Although the cost of traversing a rope is linear
per operation, people who think about mutating a rope pay the price -- and the
reason people think about modifying the contents of a rope is because it's
possible to do so.

This paper I attempt to address the issue direct at the heart and try to present
a theory (or theorem, or hypothesis) of how a string data structure should be
implemented, what the semantics of this implementation should be, and how it is
going to be arguably better than ``std::string``.

Along with that goal, here are other problems I have with the state of string
processing in C++ which I also aim to address:

* **Contiguity** -- as it is at the moment, virtually every implementation of
  ``std::string`` makes it contiguous in memory. This is bad for many reasons
  and one of these reasons is the fact that memory models and memory managers in
  modern operating systems (and platforms) deal with memory in chunk-sized
  pages. Requiring that a string be actually contiguous in memory so that a
  call to ``c_str()`` or ``data()`` would return a valid pointer that represents
  the raw bytes in memory is actually a bad thing.

* **Efficiency** -- although ``std::string`` is as efficient as an
  ``std::vector`` when it comes to traversal and operations that require random
  access to any part of the string, both of these data types are not equipped to
  grow or shrink effectively. This is mostly because they are mutable, and
  mostly because the C++ allocators are not able to handle shrinking and growing
  of already-allocated memory. [#]_

.. [#] There is a proposal called allocplus which aims to solve this issue:
   http://www.drivehq.com/web/igaztanaga/allocplus/

* **Simplicity** -- ``std::string`` as it stands has a lot of member functions
  which can be implemented as external algorithms. A lot of the member functions
  also result in temporary ``std::string`` instances which is a wasteful
  practice especially if the implementation doesn't use copy-on-write
  techniques. Even if the string does implement copy-on-write techniques to
  mitigate the cost of this temporary string, the C++0x standard will disallow
  this technique from being considered standard conforming. [#]_

.. [#] This statement is mostly feedback from those who follow the standard
   development closely; TODO: link to paper missing.

* **Separation of Concerns** -- ``std::string`` involves itself with three
  things: construction, manipulation, and storage of byte sequences. In a purely
  design-based perspective this is usually a sign of bad design. Because of
  these independent issues being clobbered together in a single class even if 
  it  is a template lends itself to really bad pathological case performance and
  a proliferation of sub-optimal implementations. This paper will attempt to
  decouple the notions of string construction and storage into appropriate 
  types and concepts. I will also go into a short explanation of why string
  manipulation should be discouraged.

These above four issues are what the paper will focus on. The paper will also
present a way of performing string-specific transformations that are lazily
applied to maximize performance gains at runtime. I will present a simple
embedded domain-specific language to make string operations more naturally and
efficiently modeled using purely C++ semantics and operators.

Propositions
------------

The paper will list down all propositions that lead up to the organized C++
string theory. First we build up on the rationale for why a string should be
immutable and what kinds of algorithms we can apply in strings. At the end we
propose a sort of string calculus which allows us to perform optimizations,
calculate certain metrics, and guide the implementation in doing what it has to
do.

Proposition 1: Strings should be immutable.
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Once a string is built the string cannot be changed at any time during its
lifetime. There are many reasons for this:

#. The underlying storage can be made specifically non-contiguous. This allows
   for more efficient use of memory for paging, alignment, and for avoiding 
   memory fragmentation.

#. Reference counting or garbage collection may be the means by which string
   block lifetimes are managed. Using a suitably efficient allocator or
   potentially a garbage collecting block allocation strategy, the memory
   management of string blocks can be made efficient and customizable according
   to the particular needs of the situation.

#. Because of the guarantee of immutability, it will play nicely with modern
   multi-core and non-uniform-memory-architecture (NUMA) CPUs for cache
   coherency concerns as well as playing nicely with an OS-level virtual memory
   manager.

#. An immutable string is thread-safe by design.

#. Removing the mutation functions allowed by the ``std::string`` implementation
   actually greatly simplifies the interface of a string type.

These are some of the technical reasons why an immutable string is better than a
mutable string like ``std::string``. The following are more conceptual reasons
for making strings immutable:

* Removing the notion of mutation from the equation forces algorithm
  implementors to look at more idiomatic means of building new strings from
  existing strings.

* By explicitly making operations on strings algorithms, the burden of covering
  the vast field of string algorithms is much more manageable and extensible.
  This means new algorithms that operate on strings will all abide by the same
  interface instead of having some algorithms as members of the type and having
  others as external function implementations.

* Making immutable strings cheap to copy and return, even without move semantics
  an immutable string implementation will greatly simplify interfaces that will
  deal with these strings.

Proposition 2: Operations on strings should be lazy.
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

As Prop. 1 suggests, a string once created can't be changed but it can be
operated upon. There are a number of fundamental string algorithms that we
define and this proposition suggests that these operations be delayed until the
resulting data is actually required.

Before we define the operations, let's define the meaning of lazily evaluated
operations. [#]_ To do this let's show what a *strict* or *immediate* operation
looks like. As an example let's define a substring operation:

.. [#] For a more in-depth discussion on lazy evaluation, see
   http://en.wikipedia.org/wiki/Lazy_evaluation

.. code-block:: c++

    template <class String>
    String substr(String s, size_t offset, size_t length) {
        // find the substring of s and then...
        typename String::iterator begin = s.begin();
        advance(begin, offset);
        typename String::iterator end = begin;
        advance(end, length);
        strings::builder builder;
        builder << strings::range(begin, end);
        String substring = builder.str();
        return substring;
    }

This strict version will build a new string immediately from a given string.
What then happens when you perform a nested substring operation like:

.. code-block:: c++
    
    string s = substr(substr(a, 10, 10), 5, 5);

In the strict implementation, this would mean building two strings from ranges
of the same string. If constructing a ``builder`` takes time and resources, then
that would add to the cost of the substring operation.

If we look closely at the nested substring operations, we can actually make this
more optimal by just saying:

.. code-block:: c++

    string s = substr(a, 15, 5);

By making the substr operation lazy, we can effectively just wrap the string and
the operation information when the data is actually required. One implementation
of the substring operation would look like this: [#]_

.. [#] This could also be achieved with Boost.Proto but for the sake of
   discussion, an expository implementation is presented. A Boost.Proto based
   solution can actually make more sophisticated optimizations possible without
   changing the semantics of the expression.

.. code-block:: c++
    
    template <class String>
    struct substr {
        String s;
        size_t offset, length;

        substr(String s, size_t offset, size_t length) 
        : s(s), offset(offset), length(length) {}

        substr(substr const & s, size_t offset, size_t length)
        : s(s.source()), offset(s.offset+offset)
        , length(min(s.length-offset, length)) {}

        typedef typename String::iterator iterator;
        // ...
        iterator begin() {
            iterator b = s.begin();
            advance(b, offset);
            return b;
        }

        iterator end() {
            iterator e = begin();
            advance(e, length);
            return e;
        }

        operator string () const {
            builder b;
            b << range(begin(), end());
            return b.str();
        }
    };

This implementation relies on the cheap to copy strings and is a "cheap" way of
doing optimizing operation layers.

As mentioned earlier there are different operations defined on strings. These
fundamental operations are:

* **Concatenation** -- by default concatenation should be lazy. In a similar
  fashion above, a concatenation operator can build a list of strings to
  concatenate (or use more clever techniques like linear inheritance) and then
  build the final string at the point of conversion.

* **Substring** -- as illustrated above.

* **Filtration** -- by removing certain matching characters (black list filter) 
  or permitting certain characters (white list filter).

* **Tokenization** -- by segmenting a string according to individual tokens
  delimited by certain provided characters.

* **Search/Pattern Matching** -- the process of providing a pattern (potentially
  regular expressions) and returning matching substrings or ranges.

There may be other operations but these listed above are considered fundamental.

Proposition 3: Building strings does not change strings.
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Because of Prop. 1 once strings have been built they cannot be changed. This
proposition reinforces this by suggesting that if you're building strings from
other strings, that you cannot change the component strings. This also implies
that since strings are immutable, it's okay and preferred that the original
string from which a new string is made will be "referred to" in the creation
process.

For this proposition we borrow from the interface provided by the
``std::ostringstream`` specification. This interface is very extensible even for
user-provided types, and can very well be used for the interface of a builder
type.

The builder type can then depend on the following elements:

* A suitable block allocator implementation. It is expected that an allocator
  that supports growing/shrinking of blocks would be used. [#]_

.. [#] See allocplus: http://www.drivehq.com/web/igaztanaga/allocplus/

* A suitably performance-sensitive implementation of a B-tree [#]_, AVL, or
  Red-Black tree for defining the concatenation of string blocks.

.. [#] See Boost.BTree: https://github.com/Beman/Boost-Btree

* A reference-counted or garbage collected block type. These storage blocks are
  then referred to directly by the concatenation trees that define a string.

The builder and string implementations will be tied in a manner that will be
inseparable -- largely because a concatenation tree will be portable and
referred to by string objects. Concatenating two strings will mean creating a
new concatenation tree for that given string. The builder class can also choose
to optimize the storage of two strings that when concatenated fit in a single
block that is grown/shrunk appropriately. [#]_

.. [#] Concatenation trees are not a new concept. The implementors of the
   ``rope`` data structures mention concatenation trees already, but they don't
   optimize the storage of string blocks in the C++ implementation. See
   http://www.cs.ubc.ca/local/reading/proceedings/spe91-95/spe/vol25/issue12/spe986.pdf
   for more information.

The performance characteristic of using blocks allows strings that fit in a
single block to have the same (if not better) performance profile as that of a
regular ``std::string`` but is much cheaper to copy -- because instances of the
same string can refer to the same concatenation tree -- and are already by
design thread-safe (because they are immutable).

Proposition 4: Strings are values.
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This proposition demands value semantics from the string. This means a string
should behave like any primitive type with the exception of mutation of the
underlying data. A string object is thus a proxy for the real string which it
represents. The suggestion is to allow the following:

* Default construction of an empty string.

* Assignment to a string: make this string object equal with another string
  object.

* Comparing two strings for equality: check if these two string objects are
  equal.

* Optionally, swappable.

As a value it should behave as a value, which means it can be copied and
referred to following the same rules of other values.

Proposition 5: String interpretation is composition.
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The proposition provides for the interpretation of data encapsulated in a string
to be something to be built around a string. This is a corollary to Prop. 2
where since operations on strings are not performed until actually necessary,
when we actually view a string through iterators or through conversions we think
of them as composing either a new type or layering operations.

When composing functions in math, we deal with certain function notation and
function application semantics. The ``composition`` operator (or the 'circle'
operator) is defined as the following::

    f(x) = ...
    g(x) = ...

    f o g = f(g(x))

This means, an interpretation of a string is a composition of a string and an
interpretation function (which in C++ would be modeled as a type).

Proposition 6: Encoding is extrinsic to strings.
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

A string has no intrinsic encoding. Because a string is a value according to
Prop. 4 and that Prop. 3 implies that once a string is built from other strings,
that an encoding cannot be enforced as part of the type. Further, as encoding is
a matter of interpreting a string, given Prop. 5 an encoding is therefore a
composition of an encoding operation and a string.

For dealing with data that is already immutable given by Prop. 1, what we need
is really a means of building strings as given by Prop. 3 that allows us to view
the string in a given encoding. By not assuming that a string has any inherent
encoding it allows algorithm writers to develop truly generic algorithms that
deal with strings. Even if encoding was a matter of transforming characters in
an immutable string, the opportunity of defining how the contents of the string
are laid out should fall as a responsibility of the builder as in Prop. 3.

By already having an opaque sequence of characters as an underlying storage,
what we can do is apply a view on the string by composing the encoding view with
an underlying string. The interface of the view would be similar to the
following template:

.. code-block:: c++
    
    template <class Encoding>
    struct view {
        string data;
        
        explicit view(string data);

        view(view const &); // copy constructible

        view & operator=(view other); // assignable

        typedef typename character<Encoding>::type value_type;

        struct iterator {
            typename value_type value_type; // depending on the encoding
            // ... and all required iterator interface definitions
            // while the iterator will not give mutable access to
            // the underlying type by having references refer to a
            // cached copy of the data
            // ... and the iterator type shall model a random access
            // iterator
        };

        iterator begin() const {
            return iterator(data);
        }

        iterator end() const {
            return iterator(data);
        }

        string raw() const {
            return data; // return a value
        }

    };

Notice that in the interface there are no string-specific member functions
defined. This is so that algorithms will only have to deal with the range as
exposed by the interface. Therefore there is no way for the view to create new
strings as it is meant to behave the same as an immutable string as far as the
interface and implied semantics is concerned.

Proposition 7: Algorithms operate on strings, but strings don't have algorithms.
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

All algorithms that deal with strings should deal with ranges. There should be
no member functions part of the string interface that imply that somehow a
string accepts messages, performs operations, or has an intrinsic capability
aside from being a string.

This reinforces Prop. 4 and Prop. 1 and is meant to emphasize that algorithms
apply to values. As hinted above in Proposition 6, an immutable string with an
assumed encoding as composed with a view shall define the character type as
defined by the encoding scheme. This then means that the interpretation of
values yielded by a view to the edges, meaning on the user's code that is
supposed to deal with the values.

Let's take an example: transcoding of a string viewed as UTF-32 into a UTF-8
``std::string`` instance.

.. code-block:: c++

    typedef encoded_builder<utf32_encoding> builder;
    builder instance;
    instance << "This should be encoded in UTF-32, with special characters.";
    builder::string_type utf32_encoded = instance.string();
    std::string utf8_encoded;
    transcode(utf32_encoded, std::back_inserter(utf8_encoded), utf8_encoding());

As per Prop. 6, the default view for the string encoded by a string builder
would only be known by the builder. The ``encoded_builder`` template can then
look like this (partially):

.. code-block:: c++

    template <class Encoding, class Allocator = block_allocator>
    struct encoded_builder : builder_base {

        typedef view<Encoding> string_type;

        string_type string() {
            return builder_base.string(buffer);
        }

    private:

        block_buffer<Allocator> buffer;

        // ... 
        // private functions accessible to the
        // namespace-level operator<< overload
        // implementations
        // ...
    };

By tying the encoding of a string with the building of the string, we should be
able to write algorithms that deal directly with the string abstraction and
specialize on the encoding specifics. With this scheme it would be trivial to
implement a ``null_encoding`` which treats data pushed into builders to store
the data "as-is" and build strings that act as immutable byte sequences.

Proposition 8: Contiguity is not a property, it's a result of an algorithm.
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

As an extension of Prop. 6 that makes encoding an extrinsic trait applied to
strings and Prop. 5 suggesting that interpreting a string is a matter of
composition, this proposition recognizes that contiguity is an important
aspect for strings that are interoperable with existing C-string based APIs and
suggests the preferred way for immutable and explicitly non-contiguous strings
to be made into something that is contiguous.

The algorithm we present is called *linearization* which is the process of
turning anything that is not explicitly contiguous into something that is
explicitly contiguous. A linearization algorithm is expected to traverse the
entire string and renders it into a bounded contiguous buffer in linear time
complexity.

One popular algorithm that can potentially perform linearization is
``std::copy`` if the supplied output iterator is tied to a contiguous buffer
like ``std::array``, ``std::vector``, or ``char *``. Here though we present an
algorithm that requires a MutableContiguousBufferIterator concept which has the
following semantics:

.. code-block:: c++
   
    // TODO define the semantics of the MutableContiguousBufferIterator concept
    // here!

The algorithm is called (aptly) linearize which takes a string, and a
MutableContiguousBufferIterator as parameters.

.. code-block:: c++

    template <class String, class MutableContiguousBufferIterator>
    MutableContiguousBufferIterator
    linearize(String s, MutableContiguousBufferIterator b) {
        typename String::iterator c = s.begin(),
                                  d = s.end();
        return std::copy(c, d, b);
    }

Interface Specifications
------------------------

TODO: write this down!

Implementation Details
----------------------

TODO: write this down! And... Implement it! :D


