A Selectors3 parser based on Syntax3
####################################

First revision.
Simon Sapin, 2012-06-09

Introduction
============

This is an attempt at a formal specification for Selectors Level 3
parser based on the `css3-syntax
<http://dev.w3.org/csswg/css3-syntax/>`_ state-machine (in its
2012-06-09 editor’s draft) rather than the CSS 2.1 Core Grammar.

It is proposal for replacing the grammar in `section 10
<http://www.w3.org/TR/css3-selectors/#w3cselgrammar>`_
of the Selectors spec.

The parser in this document accepts a super-set of allowed Level 3
selectors: the list of allow pseudo-elements and pseudo-classes is
deliberately not included. This allows eg. css3-lists to define the
``::marker`` pseudo-element without changing the "main" parser.

A functional pseudo-class is represented by its name and unparsed list
of arguments. The rules for parsing the arguments are defined
separately, case-by-case.

All strings are Unicode codepoints, unless otherwise specified.

.. _original string:

"The original string" refers to the tokenizer’s input. It is in
CSS syntax and can thus contain comments or backslash-escapes. These
are resolved by the tokenizer.


.. contents::

Input
=====

The input for this parser is assumed to have already gone through the
tokenizer and tree construction steps for Syntax3. It matches what
Syntax3 puts in the selector of a style rule. An additional procedure
should probably be added to Syntax3 for the tree construction of a
stand-alone selector, as found for example in getElementsBySelector().

.. _sequence of regrouped tokens:

The input is a sequence of "regrouped tokens".
`(TODO: bikeshed the name. Maybe something with 'tree'?)`
Regrouped tokens are close to but slightly higher-level than the tokens
produced by the Syntax3 tokenizer.

.. _regrouped token:

A regrouped token is either:

* An identifier, at-keyword, hash, string, url, delim, number,
  percentage, dimension, unicode-range, whitespace, colon or
  semicolon token.
* A block
* A function

Blocks have:

* a type: '{', '[' or '('
* content: a `sequence of regrouped tokens`_. Does not include the
  opening or closing tokens for this block.

Functions have:

* a name: a string
* arguments (content): a `sequence of regrouped tokens`_.
  Does not include the opening or closing parentheses.

Regrouped tokens are never function, bad-string, bad-url, cdo, cdc,
open-brace, close-brace, open-paren, close-paren, open-bracket
or close-bracket tokens. These either trigger a parse error
(in which case there is no selector to parse) or are turned into
regrouped blocks or functions.

Although they are never allowed in Level 3 selectors, some tokens
like url and unicode-range are included to keep this definition
as wide as possible for future levels.

Side note:
    An explicitly-defined concept like regrouped tokens can be useful
    as Syntax3’s output not only for selectors, but also for property
    values and unparsed at-rules.


Namespaces
----------

In addition to a `sequence of regrouped tokens`_, the input for this
parser is made of a (possibly empty) mapping of namespace prefixes
to URIs and an optional URI for the default namespace.

In CSS, namespace are declared with `@namespace
<http://www.w3.org/TR/css3-namespace/>`_ rules.


Output
======

If the selector is valid, the output of this parser is a tree
of objects. The root of the tree is always a group of selector.

A group of selector object is a list of one ore more simple selectors.

A selector object is made of:

* A `combined selector`_ or `sequence of simple selectors`_
* An optional pseudo-element name (a string)


.. _combined selector:

A combined selector is made of:

* On its left: a `combined selector`_ or `sequence of simple selectors`_
* A combinator: one of descendant, child, adjacent sibling or
  general sibling
* On its right: a `sequence of simple selectors`_


.. _simple selector:

A simple selector is either a `type selector`_, an
`universal selector`_, a `class selector`_, an `attribute selector`_,
a `simple pseudo-class`_, a `functional pseudo-class`_ or a
`negation pseudo-class`_.


.. _sequence of simple selectors:

A sequence of simple selectors must starts with a type selector or
an universal selector. A type selector or universal selectors
must be first in a sequence of simple selectors.
An implicit universal selector in the `original string`_ will be
explicit in the parsed sequence of simple selectors.


.. _namespace object:

A namespace object are made of:

* A namespace "type": one of ``URI``, ``any`` or ``none``
* If the type is ``URI``, the URI (a string)

``ns|E`` in the `original string`_ becomes ``URI`` (or gives a parse
error if the prefix is not declared), ``*|E`` becomes ``any``, ``|E``
becomes ``none``, and ``E`` becomes ``URI`` if a default namespace is
declared, ``any`` otherwise.


.. _type selector:

Type selectors are made of:

* A `namespace object`_
* A type name (a string)


.. _universal selector:

Universal selectors are made of:

* A `namespace object`_


.. _class selector:

Class selectors are made of:

* A class name (a string)


.. _ID selector:

ID selectors are made of:

* An identifier (a string)


.. _attribute selector:

Attribute selectors are made of:

* A `namespace object`_
* An attribute name (a string)
* An operator: one of ``exists``, ``=``, ``~=``, ``|=``, ``^=``, ``$=``
  or ``*=``
* If the operator is not ``exists``, a value (a string)


.. _simple pseudo-class:

Simple pseudo-classes are made of:

* A name (a string)


.. _functional pseudo-class:

Functional pseudo-classes are made of:

* A name (a string)
* Its arguments. The shape/type is defined for each pseudo-class.
  Level 3 pseudo-classes are defined at the end of this document.


.. _negation pseudo-class:

Negation pseudo-classes are made of:

* A negated selector. (a `simple selector`_ that is not itself
  a negation)

**Issue 1:**
    These definitions encode the constraint that a pseudo-element
    can only be last. Should they be more general, in case future
    levels want to relax the constraint?


Invalid selector
----------------

If at any point an invalid selector is encountered, the parser is
aborted and there is no output/result tree. It is up to the host
language to define what happens to invalid selectors.

For CSS style rules, it up to the Syntax3 tree construction to make
sure that an invalid selector and its declaration block are completely
consumed and ignored.


Selector parsing
================

Just like "raw" tokens, a `sequence of regrouped tokens`_ can be
consumed item-by-item with an implicit iterator/index. When the end of
a sequence has been reached, consuming it further yields eof tokens.

Note that a eof token while consuming the content of a block or
a function marks the end of the block or function, not the end of
the selector. Likewise, eof while consuming the input sequence
marks the end of the selector, not that of any larger unit (like
a CSS stylesheet) where the selector was read.


**TODO: the actual state-machine-based parser.**


Parsing of functional pseudo-classes arguments
==============================================

Each functional pseudo-class has a specialized parser for its arguments.
These parsers can either make the selector invalid or return the
arguments in a higher-level form.

The input is the arguments of the function object that represents
the pseudo-class, with whitespace tokens removed at the start and end
of the sequence. It is a `sequence of regrouped tokens`_.

:lang()
-------

**Output:** a string

If there is exactly one argument and that argument is an ident token,
return the token’s value. Otherwise, the selector is invalid.

**Issue 2:**
    Are string tokens allowed instead of ident?


:nth-child(), :nth-last-child(), :nth-of-type() and :nth-last-of-type()
-----------------------------------------------------------------------

**Output:** a pair (a, b) of integers

**Internal state:** a, b (integers), negative-b (flag, initially unset)

This should match the grammar defined in the current level 3 spec::

    nth
      : S* [ ['-'|'+']? INTEGER? {N} [ S* ['-'|'+'] S* INTEGER ]? |
             ['-'|'+']? INTEGER | {O}{D}{D} | {E}{V}{E}{N} ] S*
      ;

**Issue 3:**
    Whitespace is allowed on either side of b’s sign, but not between
    a and its sign (if any). Is this what we want?
    This seems consistent with the "whitespace" examples in the spec.

Consume the arguments one-by-one, and start in `nth-start mode`_.

nth-start mode
..............

Consume the next argument.

ident token with the value 'even'
    Set a to 2, b to 0. Switch to the `nth-end mode`_.

ident token with the value 'odd'
    Set a to 2, b to 1. Switch to the `nth-end mode`_.

ident token with the value 'n'
    Set a to 1. Switch to the `nth-after-n mode`_.

ident token with the value '-n'
    Set a to -1. Switch to the `nth-after-n mode`_.

ident token with the value 'n-'
    Set a to 1. Set the negative-b flag. Switch to the
    `nth-after-b-sign mode`_.

ident token with the value '-n-'
    Set a to -1. Set the negative-b flag. Switch to the
    `nth-after-b-sign mode`_.

dimension token with the integer flag and the unit 'n'
    Set a to the token’s value. Switch to the `nth-after-n mode`_.

dimension token with the integer flag and the unit 'n-'
    Set a to the token’s value. Set the negative-b flag. Switch to the
    `nth-after-b-sign mode`_.

dimension token with the integer flag and unit that matches 'n-[0-9]+'
    Set a to the token’s value. Set b to the token’s unit parsed
    as a decimal integer, after removing the initial 'n'.
    Switch to the `nth-end mode`_.

number token with the integer flag
    Set a to 0. Set b to the token’s value. Switch to the
    `nth-end mode`_.

anything else
    The selector is invalid.

nth-after-n mode
................

Consume the next argument.

eof token
    Set b to 0. Return (a, b)

whitespace token
    Do nothing. Remain in this mode.

delim token with the value '+'
    Switch to the `nth-after-b-sign mode`_.

delim token with the value '-'
    Set the negative-b flag. Switch to the `nth-after-b-sign mode`_.

number token with the integer flag
    Set b to the token’s value. Switch to the `nth-end mode`_.

anything else
    The selector is invalid.

nth-after-b-sign mode
.....................

Consume the next argument.

whitespace token
    Do nothing. Remain in this mode.

number token with the integer flag and a representation that does not start with a '-' or a '+'
    Set b to the opposite of the token’s value if negative-b is set,
    to the token’s value otherwise.
    Switch to the `nth-end mode`_.

anything else
    The selector is invalid.

nth-end mode
............

Consume the next argument.

eof token
    Return (a, b)

anything else
    The selector is invalid
