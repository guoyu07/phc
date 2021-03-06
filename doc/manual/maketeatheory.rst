.. _maketeatheory:

:program:`Maketea` Theory
=========================

Introduction
------------

:program:`maketea` is `available
separately <http://maketea.googlecode.com>`_ to |phc|. Based on a grammar definition of a language, it
generates a C++ hierarchy for the corresponding abstract syntax tree, a tree
transformation and visitor API, and deep cloning, deep equality and pattern
matching on the AST. In this document we describe the grammar formalism used by
|phc|, how a C++ class structure is derived from such a grammar, and explains
how the tree transformation API is generated. The generated code itself is
explained in :ref:`apioverview`. 

The Grammar Formalism
---------------------

The style of grammar formalism used by :program:`maketea` is sometimes referred
to as an "object oriented" context free grammar.  It facilitates a trivial and
reliable mapping between the grammar (:ref:`grammar`, and the actual
(C++) abstract syntax tree (AST) that is generated by the |phc| parser.  

We make a distinction between three types of symbols: *non-terminal* symbols,
*terminal symbols* and *markers*.  Non-terminal symbols have the same function
in our formalism as in the usual BNF formalism, and will not be further
explained. We denote non-terminal symbols in lower case in the
grammar (e.g., ``expr``).  

The distinction between terminal symbols and markers is non-standard.  Markers
have no semantic value other than their presence; an example is ``"abstract"``.
Thus, the semantic value of a marker is a boolean value; it is either there, or
it is not (note that this is different from a symbol such as the semi-colon,
which has **no** semantic value whatsoever, and thus does not need to be
included in an abstract syntax tree).  Conversely, the semantic value of a
*terminal symbol* is an arbitrary value; an example is :class:`CLASS_NAME` (the
structure of a terminal symbol may be defined by a regular expression; this is
irrelevant as far as the abstract grammar is concerned). We denote markers in
quotes (``"abstract"``), and terminal symbols in capitals
(:class:`CLASS_NAME`).  

Each non-terminal symbol ``aa`` will have a single production in the grammar.
Instances of ``aa`` in the AST will be represented by a class called ``Aa``.
The attributes of ``Aa`` will depend on the production for ``aa`` (see below). 

A terminal symbol ``xx`` will be represented by a class ``XX``. Every token
class gets an attribute called ``value``. The type of this attribute depends on
the token; for most tokens it is ``String*`` (this is the default); however, if
the grammar explicitely specifies a type for the value (in angular brackets,
for example ``REAL<double>``), this overrides the default. If the type of the
``value`` attribute it set to be empty, the token class does not get a value.

Finally, a marker will not be represented by a specialised class.  Instead, a
marker ``"foo"`` may **only** appear as an optional symbol in a production rule
(``a ::= ... "foo"? ...``), and will appear as a boolean attribute ``is_foo``
in the class representing ``aa`` (``Aa``).  

There are only two types of rules in the grammar. The first is the simplest,
and list a number of alternatives for a non-terminal symbol ``aa``:

.. sourcecode:: haskell

   aa ::= b | c | ... | z


.. todo:: 
   
   these can be Terminals too

Here, each of ``b``, ``c``, ..., ``z`` must be a single non-terminal symbol.
This rule results in a (usually) empty ``class Aa {}``, which acts as a
superclass for the classes for ``b``, ``c``, ..., ``z``. This reflects the
semantics of the rule (a ``b`` **is** an ``a``); if there are
multiple rules ``aa ::= c|...``, ``b ::= c|...``, class ``C`` will inherit from
both ``Aa`` and ``B``. This type of rule is exemplified by the production for
``Statement`` in the grammar. There is one additional requirement for
disjunction rules, which will be explained in the section on context
resolution, below.  

The second type is the most common: 

.. sourcecode:: haskell

   aa ::= b c ... z


In this rule, each of the ``b``, ``c``, ..., ``z`` is an arbitrary symbol
(non-terminal, terminal or marker), which may be optional (``b?``) or repeated
(``b*`` or ``b+``). This type of rule must not include any disjunctions
(``b|c``), and only single symbols can be repeated (no grouping).  If a symbol
``b`` can be repeated, it will be represented by a specialised list class
``B_list`` (which inherits from the STL ``list`` class) in the tree. In
addition, the symbols may be labeled (``label:symbol``). This does not add to
the grammar structure, but explains the purpose of the symbol in the rule, and
will be used for the name of the attribute of the corresponding class.  The
default name for each class attribute depends on the corresponding type: an
attribute of type :class:`Variable_name`  will be called ``variable_name``. The
default name for an attribute of type ``Foo_list`` will be **foos**.  However,
as mentioned above, this can be overridden by specifying a label.  

As an example, consider the rule for :class:`Variable` in the grammar.

.. sourcecode:: haskell

   Expr ::= ... | Variable | ... ;
   Variable ::= Target? Variable_name array_indices:Expr?* ;


A :class:`Variable` is an :class:`Expr`, so that :class:`Variable` is represented by the class
shown below.

.. todo::

   I removed a discuss about optional attributes, since string_index isnt
   supported in variable anymore. Does this need to be discussed?

.. sourcecode:: c++

   class Variable : virtual public Expr
   {
   public:
      Target* target;
      Variable_name* variable_name;
      Expr_list* array_indices;
   }


A final note on combining ``*`` and ``?``. The construct ``(a*)?`` denotes an
optional list of ``a``\s. Thus, it will be represented by an ``A_list``. If a
list is specified, but empty, the list will simply contain no elements. If the
list is not specified at all, the list will be NULL. This is used, for example,
to distinguish between methods that contain no statements and abstract methods.
Similarly, ``(a?)*`` is a (non-optional) list of optional ``a``\s. Thus, this
is a list, but elements of the list may be NULL.  This is used for example to
denote empty array indices (``a[]``) in the rule for ``Variable``.  

.. _contextresolution:

Context Resolution
------------------

We also derive the tree visitor API and tree transformation API from the
grammar. The tree visitor API is very simple to derive; see the
:ref:`apioverview` for an explanation. The tree transformation API however is
slightly more difficult to derive. The problem is to decide the signatures for
the transform methods, or in other words, what can transform into what? For
example, in the |phc| grammar for PHP, the transform for an if-statement should
be allowed return a list of statements of any kind (because it is safe to
replace an if-statement by a list of statements).  Similarly, a binary operator
should be allowed return any other expression (but not a list of them). For
reasons that will become clear very soon, we call the process of deciding these
signatures "context resolution".


Contexts
********

A context is essentially a use of a symbol somewhere in a (concrete) rule in
the grammar.  There are four possibilities. Consider: 

.. sourcecode:: haskell

   concrete1 ::= ... 
   concrete2 ::= ...
   concrete3 ::= ...
   concrete4 ::= ...
   concrete5 ::= ...
   concrete6 ::= ...
   abstract1 ::= concrete3 | concrete4
   abstract2 ::= concrete5 | concrete6
      
   some_concrete_rule ::= concrete1 concrete2* abstract1 abstract2* 


then, based on the rule for ``some_concrete_rule``, ``concrete1`` occurs in the
context ``(concrete1,concrete1,Single)`` - i.e., as a single instance of
itself, concrete2 occurs in the context ``(concrete2,concrete2,List)``, i.e.
as a list of instances of itself. The use of the ``abstract1`` class leads to a
number of contexts: 

.. sourcecode:: haskell

   (abstract1,abstract1,Single)
   (concrete3,abstract1,Single)
   (concrete4,abstract1,Single)


And finally, the use of ``abstract2*`` yields to the contexts 

.. sourcecode:: haskell

   (abstract2,abstract2,List)
   (concrete5,abstract2,List)
   (concrete6,abstract2,List)


These contexts essentially mean that an instance of ``concrete5`` can be
replaced by any number of any (concrete) instance of ``"abstract2"``. 


Reducing Contexts
-----------------

If there are two or more conflicting contexts for a single symbol, we must
resolve the contexts to their most specific (restrictive) form.  For instance,
for the |phc| grammar, this yields 

.. sourcecode:: haskell

   (if,statement,List)
   (CLASS_NAME,CLASS_NAME,Single)
   (INTERFACE_NAME,INTERFACE_NAME,Single)


So, a context is a triplet ``(symbol,symbol,multiplicity)``, where the symbols
are terminal or non-terminal symbols, and the multiplicity is either
``Single``, ``Optional``, ``List``, ``OptionalList`` or ``ListOptional`` (list
of optionals).  When reducing two contexts (``a``, ``b``, ``c``)
(``a'``, ``b'``, ``c'``), we take the meet of ``b`` and ``b'`` (that is, the most
general common subclass of ``b`` and ``b'``, where more general means higher up
in the inheritance hierarchy), and opt for the most restrictive Multiplicity
(Single over Optional, Single over List, etc.). The general idea is that we
want the most permissive context for a non-terminal that is still safe: if it
is safe to replace an ``a`` by a list of ``b``\s **everywhere** in a tree, the
context we want for ``a`` is (``a``, ``b``, list). 

To see the reason for taking the meet, consider this fragment of the |phc|
grammar:

.. sourcecode:: haskell

   Expr ::= ... | BOOL
   Cast ::= CAST Expr
   Method_invocation ::= Target ...
   Target ::= Expr | CLASS_NAME


The use of "expr" in the rule for cast leads to the context
``(BOOL,expr,Single)`` The use of "target" in the rule for method_invocation
leads to the context ``(BOOL,target,Single)``. By taking the meet of "expr" and
"target", this gives the context ``(BOOL,expr,Single)``. This means that it is
always safe to replace a boolean by any other expression (but it is not always
safe to replace a boolean by any other *target*).
	
In the case of :class:`CLASS_NAME`, we have the contexts

.. sourcecode:: haskell

   (CLASS_NAME,class_name,Single)
   (CLASS_NAME,target,Single)


The meet of class_name and target does not exist; hence this gives the context
	
.. sourcecode:: haskell

   (CLASS_NAME,CLASS_NAME,Single)


That is, the only safe transformation for :class:`CLASS_NAME` is from
:class:`CLASS_NAME` to :class:`CLASS_NAME`.

To be precise about the "most specific" multiplicity, here is a Haskell
definition that returns the meet of two multiplicities:

.. sourcecode:: haskell

   meet_mult :: Multiplicity -> Multiplicity -> Multiplicity
   meet_mult a b | a == b = a
   meet_mult Single _ = Single  
   meet_mult List Optional = Single 
   meet_mult List OptList = List
   meet_mult List ListOpt = List
   meet_mult Optional OptList = Single
   meet_mult Optional ListOpt = Optional
   meet_mult OptList ListOpt = List
   meet_mult a b = meet_mult b a  -- meet is commutative


Resolution for Disjunctions
---------------------------

We cannot deal with this situation:

.. sourcecode:: haskell

   s ::= a
   a ::= b | c
   d ::= b
   e ::= c*


This grammar leads to the following contexts:

.. sourcecode:: haskell

   (a,a,Single)
   (b,a,Single)
   (b,b,Single)
   (c,a,Single)
   (c,c,List)


Resolving these contexts lead to

.. sourcecode:: haskell

   (a,a,Single)
   (b,b,Single)
   (c,c,List)


However, this is incorrect, because this indicates that an ``a`` will only be
replaced by another, single, ``a``; but a ``c`` (which is an ``a``) will in
fact return a list of ``c``\s. The problem is that the non-terminals in the rule
for ``a`` have a different multiplicity in their contexts (single for ``b``,
list for ``c``). :program:`maketea` disallows this; if this happens in a
grammar, :program:`maketea` will exit with a "cannot deal with mixed
multiplicity in disjunction" error.

Otherwise, for a rule ``a ::= b1 | b2 | ...``, if the multiplicity of ``a`` is
list, and the multiplicities of all the ``b``\s are lists, the multiplicity for
``a`` will be list; if the multiplicity of all the ``b``\s is single, the
multiplicity for ``a`` will be set to single (independent of the original
multiplicity for ``a``).
