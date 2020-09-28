Straw-man proposal for Verona strings
=====================================

Strings are one of the core components of a standard library and a good design is one of the key factors for language usability.
A good string API needs to balance several requirements:

 * Support for micro-optimisation.
   Fast-path string operations should be easy to inline and vectorise.
 * Support for macro-optimisations.
   Custom string storage models optimised for specific use cases should be possible, without the need to implement the full set of string operations.
 * Ease of use.
   The programmer should only need to care about the specific string representation when string handling operations are on a performance-critical path.
 * Fast for common operations.
   Unfortunately, the set of common operations on a string can differ significantly between applications.
 * Support all string operations.
   Some of these, such as locale-aware collation, are incredibly complex.
 * Interoperation with other systems that place constraints on string representation (e.g. system call arguments).

Many existing languages have suffered from the definition of a character changing over the lifetime of the language.
When C was introduced the consensus that a character was 8 bits was still fairly new, with older machines often supporting 6-bit characters.
The developers of C used Latin-alphabet languages and multi-byte encodings were later retrofitted to C in somewhat awkward ways, breaking the abstraction that one `char` is one `character`.

The OpenStep specification and Java came at a time when the Unicode 1.0 specification was current and there appeared to be consensus that 16 bits was sufficient to represent any character in any language.
This consensus changed in 1996 with the release of the Unicode 2.0 specification, expanding the encoding space to up to 1,112,064 code points.
The current version of the Unicode specification, 13.0, uses just over 10% of this encoding space and even with the amount wasted on emoji it seems unlikely that the total space will be exhausted and so Verona is likely to be in a in a fortunate position of being able to define a string API without the definition of a character changing substantially over the lifetime of the language.

It is incredibly unlikely; however, that a Verona string implementation will contain native Verona code that implements the full Unicode specification in an early release.
The specification of the [Unicode collation algorithm](https://www.unicode.org/reports/tr10/) alone runs to several hundred pages of text.
A string design will therefore need to be efficient when interoperating with a sandboxed C++ library, such as ICU.

Who cares about strings?
------------------------

Verona aims to be an *infrastructure programming language*.
This category encompasses target use cases such as device drivers, distributed databases, online translation engines, and so on.
Some of these will not have any user-facing component and so will use strings as identifiers (for example, dictionary keys, path components, error indicators) without the need for any localisation.
Others will use strings to present text to a human and will need rich unicode processing.
Our goal should be to support both use cases with the same set of abstractions but avoid projects in the former category needing to pay any performance or code-size penalty relative to a simple array-of-probably-ASCII-bytes string representation.

A lot of complex Verona systems are likely to include components that are at both extremes in terms of their use of strings.
We should not require a global decision about how strings are represented, nor should we penalise modules that require only simple strings.

Code units, code points, characters, and grapheme clusters
----------------------------------------------------------

Strings are, informally, sequences of characters.
This is not always a useful abstraction as the definition of a character can, in different contexts, mean:

 * A code unit, the smallest indivisible unit of a single or multi-byte encoding (e.g. `char` or `wchar_t` in C, `unichar` in Objective-C)
 * A Unicode code point, which may be stored as 1-4 octets, depending on the encoding.
 * A [grapheme cluster](https://www.unicode.org/reports/tr29/), a sequence of code points that are combined to display something that a human will consider to be a character.

Iteration over strings is therefore very complex.
Should the unit of iteration be a code unit, a code point, or a grapheme cluster?
Should iteration canonicalise any of these?
For example, é can be represented as either a single code point or as an e followed by with an accent combining diacritic, but both representations are regarded as equivalent by unicode.

Mutable and immutable strings
-----------------------------

A lot of programs depend on immutable strings.
The Java security model depends on strings being immutable and the `SecurityManager` can be bypassed if a vulnerability allows immutable string objects to be modified.
Strings are commonly used as keys in collections in most languages (including 'languages' such as POSIX, where they are used to define the filesystem namespace).
Typically, keys in collections must be immutable for as long as they are being used as keys: modifying a key without notifying the collection will affect the hash or ordering of keys and cause lookup failures.

In Verona, mutability is a property of an instance, not a class.
This allows a common class for mutable or immutable strings and provides some opportunities for string APIs.
C's overloading of `const` to mean 'thing that cannot be modified' and 'thing that this function will not modify' causes some problems in the APIs.
For example, consider the standard library function `strchr`:

```c
char * strchr(const char *s, int c);
```

This takes a `const char*` (C string) as an argument and returns a pointer into that string.
The returned pointer may point to an immutable object but it lacks a `const` qualifier.

In OpenStep, `NSMutableString` is a subclass of (immutable) `NSString`, which allows APIs to be implemented as methods on `NSString` or as functions that take `NSString` arguments.
This can lead to unsafe down-casts and the fact that a mutable string is a kind of immutable string causes bugs when mutable strings are accidentally used as immutable strings and modified unexpectedly.

In Verona, we can differentiate between `imm` parameters, where the callee may capture the argument and requires that it is never mutated, and `readonly` parameters, where the string may be mutable but the caller promises not to modify them.
Collections using strings as keys should always take an `imm & String`, whereas functions that query properties of a string should take a `readonly & String`.

Existing string implementations
-------------------------------

Java and C# both provide a standard-library string class that defines a container for UTF-16 code units.
In both cases, these are concrete types that cannot be subclassed.
Similarly, both define an immutable standard library string and builders for constructing them, with a completely distinct set of mutable string types.
This approach allows a lot of microoptimisation in the string implementation and compiler: strings are always a specific concrete type and the compiler is free to make a lot of assumptions about the behaviour of methods on strings.
It has a lot of disadvantages as well:

 - Strings and mutable strings are unrelated types and so generic code over both is difficult to write.
 - It is possible to split a string in the middle of a codepoint, leaving an invalid string, because the unit of indexing is a UTF-16 code unit.

OpenStep's `NSString` class is an abstract class that requires that subclasses implement two methods:
`-length` returns the number of UTF-16 code units in the string, `-characterAtIndex:` returns the character at a specific index.
More interestingly, most of the other methods do not use `-characterAtIndex:`, they instead use `-getCharacters:inRange:`, which copies a range of UTF-16 code units into a buffer.
The mutable subclass adds a `-replaceCharactersInRange:withString:` method, which handles insertion and deletion of an arbitrary range.
The abstract `NSString` class also provides two methods that can allow other optimisations at a higher level.
The `-fastestEncoding` method returns the encoding that this string can be converted to with the lowest overhead and the `-cStringUsingEncoding:` and `-getCString:maxLength:encoding:` methods can then request a string-allocated (and possibly internal) immutable view of the string as a C string, or a copy of the string as a C string.
By pairing these, the caller can access the character data in the string's native encoding, possibly without the need to copy, and operations on two strings that happen to be in the same encoding can be significantly faster.

This design is very flexible and allows the string to use whatever internal storage it wants (for example, ASCII / UTF-8 storage) while still allowing operations on a range of characters to do only one dynamic call.
It does not; however, allow for compile-time specialisation when the concrete type of the strings is known at compile time, which eliminates a lot of microoptimisation opportunities.

This basic pattern was extended for collections in the `NSFastEnumeration` protocol and ICU's `UText` abstraction provides a similar model.
The optimization goals for `UText` are slightly different: all of the operations on `UText` work with UTF-16 data and so the responsibility for converting to UTF-16 is delegated to individual string objects.
In `UText`, the generic text object has three fields containing a pointer to a buffer, the character at the start of the buffer, and the length of the buffer, along with a virtual function that can fill the buffer.
The generic `UText` allocator also provides an option to allocate space at the end of the object that instances can use.
The fill function can either set the data field to point to its internal storage, copy into the space at the end, or allocate additional memory to store a temporary buffer.

It is worth noting that `NSString` / `NSMutableString` and `UText` provide sufficiently similar abstractions that it is fairly trivial (under 500 lines of code) to create [bidirectional bridges between the two](https://github.com/gnustep/libs-base/blob/master/Source/GSICUString.m).

String views
------------

C++, C#, and other languages have recently introduced the idea of a view within a string, similar to a Modula-2 slice.
A string view gives a range within a string but does not duplicate the underlying storage.
If a string view is mutable then changes in it will be reflected into the underlying storage.

String views in Verona can use the region abstraction to ensure correct lifetime management.
A string view on a mutable string will be in the same region as the underlying string and it is the responsibility of that region's memory management to ensure that the region does not outlive the string view.
Alternatively, string views may be stack types that are bound to a specific region.

String views that refer to immutable strings do not introduce memory management difficulties in Verona because immutable objects are reference counted.
Even if the string view is mutable, it can be in any region without causing problems.

A string object does not, conceptually, contain any pointers, though it may contain pointers to internal storage if the characters are not stored inline.
As such, it makes sense for most concrete mutable string types to expose `iso` interfaces.
This is slightly complicated by string views, which must be in the same region as the string that they represent (or stack types bound to a specific region).

Constant strings from string literals
-------------------------------------

Baking a representation of string literals into a language causes problems in AoT-compiled languages (this is less of a problem for JIT'd or interpreted languages, where the representation can be changed when the runtime environment is upgraded).
For example, NeXT defined the Objective-C constant string representation to be an instance of the `NXConstantString` class, with a structure that contained the `isa` (class) pointer, a pointer to a C string, and the length.
This caused two significant problems over the next decade.
First, when source code stopped being ASCII, the length would be the number of bytes but the number of UTF-16 code units may differ and so implementing the `-length` method on `NXConstantString` became nontrivial.
Second, constant strings were increasingly used as dictionary keys and so needed a fast `-hash` method.
The `-hash` method on `NXConstantString` was always O(n), whereas a constant `NSString` had a field to store a fixed hash value.

In the specific case of a hash, it's not clear that it's best to have a hash for a string or a wrapper type that stores the hash.
The OpenStep decision to make every object provide a `-hash` method means that any objects that can compare equal must also use the hash function.
This effectively makes the hash function part of the ABI for strings (normally addressed by implementing `-hash` only in the base class).
This makes it impossible for different collections to use different hash functions.
C++ solves this problem by making the collections responsible for storing both the key and (if they need to cache it) the hash value computed by a function that takes a key as an argument, which is probably a cleaner solution.

Making it possible to change the default class used for immutable strings constructed from literals is still likely to be useful.
For example, lower-level code is likely to want a constant string that stores ASCII characters and a length, whereas higher-level code may want a UTF-16 or UTF-8 internal encoding and some metadata for quickly finding code unit boundaries so that asking for the *n*th code point does not require scanning *n* code points.

Necessary operations on strings
-------------------------------

Optimisation for strings can mean different things.
For immutable strings, iterating over characters (whatever that means), equality comparison, and extracting substrings form the basis of most operations.

# Iteration

ICU provides a number of kinds of iteration over strings, as does OpenStep's `NSString`.
These include iterating over UTF-16 code units, Unicode code points, grapheme clusters, words, and sentences, with the breaking rules defined by the Unicode specification.
Iterating over UTF-16 code units exists at the API layer in these designs for very different reasons.
In the case of ICU, all of the other Unicode-processing algorithms expect UTF-16.
In the case of OpenStep (and Java), the APIs were designed for UCS-2 and then had UTF-16 retrofitted: they were originally intended to use Unicode code points, not UTF-16 code units, but assumed that Unicode code points all fitted in 16 bits.

C++'s `std::string` takes the character type as a template parameter and allows iterating over this type.
It does not differentiate between code points and code units at the string class level: the string is simply a container for some fixed-width integer type and it is up to consumers to determine whether this stored integers are sufficient to store a complete code point in the consumer's encoding.
The fact that most complex C++ libraries include their own string implementation is probably an indication that this is not a good idea.

Iterating over the internal representation is useful for character set conversion.
It can also be useful for operations that don't need to be portable across all possible strings, for example hashing or locale-unaware sorting when a specific concrete string type, rather than an abstract string interface, is used.

# Random access

Random access in a string is a complex concept.
Random access in C strings or C++ `std::string` is easy because they are simply arrays of code units but it is also meaningless in the context of variable-length encodings (byte *n* in a UTF-8 string, for example, may not be the start of a valid code point, let alone a grapheme cluster).
Most random access wants to, at a minimum, index on code points and often on grapheme clusters.

# Modification

For mutable strings, inserting, deleting, and replacing substrings often need to be fast.
This typically leads to mutable strings using a twine or similar representation.
Even in-place modification of a character or a substring with a string of the same length can involve changing the number of code units in a string if the characters are replaced with different-width ones.
This provides a strong incentive to avoid mandating contiguous storage for strings and is one of the common reasons for avoiding `std::string` in C++.

# Comparison

There are four common kinds of comparison on strings:

 - Canonicalisation-unaware equality.
   Do these two strings contain the same set of Unicode code points?
 - Canonicalisation-aware equality.
   Do these two strings contain a sequence of Unicode code points that are equal under canonicalisation.
 - Locale-unaware ordered comparison.
   What is a stable ordering between these two strings that is fast to compute?
 - Locale-aware collation.
   What is the ordering between these two strings, with the comparison computed using the Unicode collation rules?

The first and third of these depend only on the strings being able to expose length and iterators that export a common character encoding.
The second and fourth require a complex unicode library and should probably be in a separate package.

# URL and path operations

Some frameworks provide operations on strings for handling path and URL data.
These are increasingly being deprecated and being moved to specific path and URL types that enforce validity and use structured storage until they must be exported to a string.
It is unlikely that a Verona string design will want to include any of these operations in the core string APIs.

Encodings
---------

Unicode defines three common serialisations, with different properties:

 - UTF-8 is the most space-efficient for most Latin alphabet languages (an Emoji) and encodes every Unicode code point in 1-4 bytes.
   The variation in length can make processing and random access indexed by code point expensive.
 - UTF-16 is the most space-efficient for CJK languages and encodes every Unicode code point to 2 or 4 bytes.
   As with UTF-8, random access requires additional metadata, though this can typically be omitted for strings in European languages, where every non-Emoji character will be 2 bytes.
 - UTF-32 is the least space-efficient encoding, using 4 bytes for every character, but has the advantage of being a fixed-length encoding (but grapheme cluster indexing remains challenging).

UTF-8 is common on a lot of Internet protocols and so may be the input or output format.
UTF-8 is typically used for system call or C library arguments on UNIX systems, whereas UTF-16 is common on Windows.

7-bit ASCII is sufficient to encode anything that can be encoded as a fixed-width UTF-8 string and is commonly sufficient for path and URL encodings and so can be worth special-case handling.
Note that, because 7-bit ASCII was mapped directly into Unicode, 7-bit ASCII is a valid Unicode encoding, though one that can store only a small (but common in English) subset of Unicode.
Lower-level code is likely to want to opt into using 7-bit ASCII by default (this is sufficiently common in C that Sun's libc has special fast-path operations that can be selected at compile time for programs that don't care about localisation).

There are two other uncommon Unicode encodings that are worth mentioning but probably not worth supporting as an internal representation for standard library strings:

 - UTF-7 is an encoding into a 7-bit character space for use with old email systems that cannot process 8-bit text.
 - UTF-EBCDIC is equivalent to UTF-8 but stores the EBCDIC character set in the 7-bit space, rather than the ASCII set.

There is no single encoding that is the best fit for all applications.
There are two key design questions related to encodings:

 - Should strings that use a non-Unicode internal encoding ever expose that across the API boundary?
   For example, should Japanese text in Big5 encoding be required to convert to Unicode for all operations that are not part of the class's internal implementation?
 - Should strings that are using a Unicode encoding expose interfaces in a *specific* unicode encoding?

*Proposal*: Standard library strings should expose interfaces in 32-bit Unicode code points and sequences thereof for grapheme clusters.
They should also be able to export or provide views in their native encoding so that operations can be optimised based on pattern matching.

Draft proposal for a core string library
----------------------------------------

The Verona standard string should define a string interface and a small set of concrete instantiations of this interface.
The core string class should implement the minimum required for generic implementation of string operations, which can be implemented either using dynamic dispatch over the string interface or as generic reifications over the concrete types.
We should make it possible to support a rich Unicode library that uses the same string objects as the standard library (and third-party libraries that use the same interface) and provides functions for iteration with unicode graphemes, locale-aware collation and other more complex operations that not all programs need.

The interface for strings would look roughly like this:

```verona
/**
 * String contains only 7-bit ASCII characters and is stored with each code
 * point in an 8-bit integer.
 */
class StringEncodingASCII
{
	typedef CharType = U8;
}
/**
 * String is stored as UTF-8-encoded text.
 */
class StringEncodingUTF8
{
	typedef CharType = U8;
}
/**
 * String is stored as UTF-16-encoded text.
 */
class StringEncodingUTF16
{
	typedef CharType = U16;
}
/**
 * String is stored as UTF-32-encoded text.
 */
class StringEncodingUTF32
{
	typedef CharType = U32;
}
/**
 * Interface for other string encodings.  Any other type can be used as a valid
 * string encoding but code that does not have any special cases for that
 * encoding will be able to treat is as unknown.
 */
interface StringEncodingCustom
{
	typedef CharType;
}

/**
 * The set of Unicode encodings.  All standard library functions that have
 * special cases for specific encodings should handle these.
 */
typedef UnicodeStringEncoding = StringEncodingASCII |
                                StringEncodingUTF8 |
                                StringEncodingUTF16 |
                                StringEncodingUTF32;
/**
 * The full set of possible string encodings, can be extended by third-party code.
 */
typedef StringEncoding = StringEncodingASCII |
                         StringEncodingUTF8 |
                         StringEncodingUTF16 |
                         StringEncodingUTF32 |
                         StringEncodingCustom;

/**
 * Character type, stores a unicode code point.
 */
typedef Unichar = U32;

// Placeholder until we have syntax for value types.
value Range
{
	U64 index;
	U64 length;
}

/**
 * Interface for string types.  Defines only the primitive operations that all
 * strings must implement.
 */
interface String
{
	/**
	 * Return the length of the string in Unicode code points.
	 */
	func length(self: readonly & String) : size_t;

	/**
	 * The encoding that this string can most easily convert to.  This will
	 * typically be the encoding that the string uses internally.
	 * Note that a return value of `StringEncodingASCII` implies
	 * `StringEncodingUTF8` for all read operations because ASCII is a strict
	 * superset of UTF-8.
	 */
	func fastest_encoding(self: readonly & String) : StringEncoding;

	/**
	 * The standard encoding that this string can most easily convert to.  This
	 * will typically be the encoding that the string uses internally but may
	 * not be for custom encodings.  For all standard-library string types,
	 * this will return the same value as `fastest_encoding`.
	 */
	func fastest_unicode_encoding(self: readonly & String) : UnicodeStringEncoding;

	/**
	 * The standard encoding that will use the smallest amount of space to
	 * store the characters.  For example, most English strings will return
	 * `StringEncodingASCII`, French will typically return
	 * `StringEncodingUTF8`, whereas CJK language strings will likely return
	 * `StringEncodingUTF16`.
	 */
	func smallest_unicode_encoding(self: readonly & String) : UnicodeStringEncoding;

	/**
	 * Access a specific character.  Returns a unicode code point for the
	 * specific index in the string.
	 */
	func apply(self: readonly & String, index: U64) : Unichar;

	/**
	 * Access a specific character.
	 */
	func apply(self: readonly & String, index: U64) : Unichar;

	/**
	 * Takes a range in the string (expressed in Unicode code points) and
	 * optionally a buffer.  Either the desired characters (in the specified
	 * encoding) are copied into that buffer, or returns an immutable array
	 * that may point to the internal storage for the string.
	 *
	 * If the caller first calls `fastest_unicode_encoding` and uses the result
	 * as the generic parameter then it will return an internal buffer for
	 * immutable strings.  This is intended to be used to build fast iterators.
	 */
	func copy_or_view_data[StringEncoding Enc](self: readonly & String,
	                                           range: Range,
	                                           buffer: Optional[Array[Enc.CharType] & mut])
	     : Array[Enc.CharType] & (mut | imm);

	/******************************************************************************
	 * Operations on mutable strings
	 *****************************************************************************/

	/**
	 * Updates a specific character. 
	 */
	func update(self: (mut | iso) & String, index: U64, c: Unichar);

	/**
	 * Replace the specified range in a string with another string.
	 */
	func update_range[String Str](self: (mut | iso) & String,
	                              range: Range,
	                              newString: Str & imm);
}
```

For discussion, the above proposal is somewhat simplified:
The `update` operation should take a `StringView`, a non-owning view of a substring.
The `String` interface should also include an operation for providing a `StringView` on a specific range.
The Verona region system makes tracking ownership of string views relatively simple:

 - By default, all mutable strings are `iso`.
 - Mutable string views are `mut`, in the same region as their owning string.
 - Immutable strings are globally immutable (`imm`).
 - Immutable string views are immutable.

This makes it easy to ensure that the lifetime of a string view is bound to the lifetime of the underlying string.
Once stack types are better defined, it should also be possible for string views to be stack types associated with the region of the string.

The minimal standard library should also include:

 - Concrete instantiations of the `String` interface, storing ASCII, UTF-8, and UTF-16.
 - Operations for finding substrings and extracting substrings as string views.
 - Operations for constructing the most size or speed-efficient concrete string type from a string literal or string view.
 - Operations for computing efficient hashes on strings.

Requirements on the language
----------------------------

Supporting a rich Verona string library imposes a few requirements on the language.
Most of these are already in progress but are captured here for completeness.

# Literal syntax

String literals in the source code should be an opaque immutable type that implements an interface that allows bytes to be iterated.
We then need lightweight syntax for passing an instance of this type to the constructor for a concrete string type.

# Generics

Most string functions will be generics that take a `String` or `StringView` interface but then match on concrete instantiations of that interface for fast paths.
The concrete type of a string will often be known to the compiler but not to the static type system.
As such, we want simple syntax for writing generics and a language-level guarantee that the compiler is free to reify and call more-specific versions if the type is known statically.
For example, consider a trivial search function:

```verona
func find_character(str : String & readonly, c : Unichar) : U64 | NotFound
{
	match (str)
	{
		AsciiString => (if (c > 127) return NotFound) else ...
		...
	}
}
```

This should be equivalent to a generic where the type used for `str` is a generic paramter.
If the compiler knows that the argument is an `AsciiString` then it should be free to substitute a version that only includes the `AsciiString` version of the `match`.

# Constant expression string interning

Strings used as keys will typically want to be interned where possible, giving an `imm` type that has cheap equality comparison.
In general, string interning is a global operation and so will need synchronisation.
In Verona, there will be one or more cowns that handle interning operations.
Many keys will be interned at compile time and we will need to have sufficiently-expressive constant expressions that we can express this.

This means that we need either to allow `when` clauses in constant expressions or we need to provide a mechanism for collecting a data structure (the set of compile-time interned strings) during constant expression evaluation and making it available to a specific cown.

# Stack types

String views can be `mut` within the region, but it will often be more convenient if string views can be on the stack.
We should consider this use case when designing stack types.