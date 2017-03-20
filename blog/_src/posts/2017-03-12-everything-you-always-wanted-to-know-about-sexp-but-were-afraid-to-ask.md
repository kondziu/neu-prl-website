    Title: Everything You Always Wanted to Know About SEXP * But Were Afraid to Ask
    Date: 2017-03-12T18:50:14
    Tags: R, by Konrad Siek
    
<!--Tags: DRAFT-->

R is an amazing language full of wonder and confusion. It has a fairly simple
definition based on function definitions and applications. There are also a
number of fun quirks like lazy argument evaluation through promises and the
ability to overload every function from `+` to, indeed, `function`. As such,
the language is really fun to play with. But it is also a real language: it has
a vast community of serious scientists who use it for serious science. It also
has a large ecosystem with a plethora of packages that extend R with specialist
functionalility. Optimizing the language is also fun, since it has much
apparent room for improvement, but its quirks make any attempts to do so
non-trivial at best, and impossible at worst.  All those things make R a unique
phenomenon.

<!--, where your fucking about/experimentation may lead to real
improvements for real people.-->

<!-- What's more though, it is used
by a vast community of serious scientists too! Research fodder! -->

<!-- _, and a plethora ofconfusing quirks that stem from it.
, but there any number of interesting quirks in it, like the ability
to overload just about any built-in definition, and passing function arguments
as promises.-->

<!--There is a snag though. The R interpreter is not necessarily easy to hack,
as I recently found out for myself, having been thrown into its depths.-->

As a PL researcher you might take a swing at hacking the R interpreter someday.
For that though, you have to get a feel for how the R interpreter does things.
This is not always as straightforward as one would like, though.  

So, I'll give
you a hand and give you a few pointers about R's SEXPs, which are one of the
first things you will encounter in the bowls of the R interpreter. I should
note though, I am not an expert on R internals (if there even is such a
thing...) just a bumbling researcher just like you, trying to figure out this
infernal machinery. Nevertheless, I have been around the block a few times and
I think I can show you a thing or two about SEXP.

<!-- , or, at least, get you
started, and hopefully save you the headache of having to figure out everything
from first principles.-->

<!--My limited exprience with it suggests this is not always as
straightforward as one would expect.-->
<!--I recently found myself thrust into the depths of the R
interpreter and quickly realized that the key to doing anything at all in its
seedy underbelly requires an understanding of the SEXP structure.-->

<!--Hopefully this post will give you a couple of hints about how to engage with
SEXPs responsibly.-->

<!-- more -->

The SEXP talk
-------------

So what is a SEXP? Conceptually, SEXPs are s-expressions (or symbolic
expressions) which take the form of a tree-like structure that describes
expressions in the the R language. For example a language expression like:

```R
{ x + y + 1}
```

is translated by the R interpreter into a tree containing SEXPs representing
language expressions, symbols, and numeric values, as follows:

```
lang
  sym '{'
  lang 
    sym '+'
    lang
      sym '+'
      sym 'x'
      sym 'y'
    real '1'
```

Additionally, things like environments, promises, and argument lists are also
represented as SEXPs in R. Futhermore, there are even cases when SEXPs are
created by the R interpreter internally out of thin air for convenience (e.g.
to use as maps or linked lists).

In other words, whenever you interact with the internals of R in any meaningful
way you will end up dealing extensively with SEXPs.

SEXP inspector
--------------

While SEXPs are omnipresent from the point of view of R, they are also
invisible to the R programmer. Nevertheless, you can ask the R interpreter to
give you fairly comprehensive information about them using the internal
`inspect` function. For instance, if we wanted to see the SEXP of the example expression we talked about before, we would do it like so:

```R
x <- 3
y <- 2
.Internal(inspect({ x + y + 1 }))
```

Actually, that would not work, since the expression would get evaluated before we passed it to inspect. So let's prevent it from doing that by running it through `substitute` like so:

```R
x <- 3
y <- 2
.Internal(inspect(substitute({ x + y + 1 })))
```

The `substitute` function will return an unevaluated parse tree for our
expression, and inspect will traverse it. In effect, the code will spit out
something like this:

```
@55989e8b9be8 06 LANGSXP g0c0 [] 
  @55989d5b0a68 01 SYMSXP g0c0 [MARK,LCK,gp=0x7000] "{" (has value)
  @55989e8b9bb0 06 LANGSXP g0c0 [] 
    @55989d5c0748 01 SYMSXP g0c0 [MARK,LCK,gp=0x7000] "+" (has value)
    @55989e8b9b78 06 LANGSXP g0c0 [] 
      @55989d5c0748 01 SYMSXP g0c0 [MARK,LCK,gp=0x7000] "+" (has value)
      @55989d61fcb0 01 SYMSXP g0c0 [MARK,NAM(2)] "x"
      @55989d7487d0 01 SYMSXP g0c0 [MARK] "y"
    @55989ed492c8 14 REALSXP g0c1 [] (len=1, tl=0) 1
```

The output will differ vastly depending on the type and content of the SEXP we are inspecting. For instance this:<sup>2</sup>

```
.Internal(inspect(substitute(function(x)x)))
```

returns this:

```
@55989d996168 06 LANGSXP g0c0 [] 
  @55989d5bae68 01 SYMSXP g0c0 [MARK,LCK,gp=0x7000] "function" (has value)
  @55989d996248 02 LISTSXP g0c0 [] 
    TAG: @55989d61fcb0 01 SYMSXP g0c0 [MARK,NAM(2)] "x"
    @55989d5b0bb8 01 SYMSXP g0c0 [MARK] "" (has value)
  @55989d61fcb0 01 SYMSXP g0c0 [MARK,NAM(2)] "x"
  @55989ee31348 13 INTSXP g0c3 [OBJ,ATT] (len=8, tl=0) 1,30,1,41,30,...
  ATTRIB:
    @55989d9961d8 02 LISTSXP g0c0 [] 
      TAG: @55989d5b9eb0 01 SYMSXP g0c0 [MARK,LCK,gp=0x4000] "srcfile" (has value)
      @55989d994d18 04 ENVSXP g0c0 [OBJ,ATT] <0x55989d994d18>
      FRAME:
	@55989d9954a0 02 LISTSXP g0c0 [] 
	  TAG: @55989da64880 01 SYMSXP g0c0 [MARK] "lines"
	  @55989eba6c68 16 STRSXP g0c1 [] (len=1, tl=0)
	    @55989ee54540 09 CHARSXP g0c4 [gp=0x60] [ASCII] [cached] ".Internal(inspect(substitute(function(x)x)))
"
	  TAG: @55989da6e830 01 SYMSXP g0c0 [MARK] "filename"
	  @55989eba6d28 16 STRSXP g0c1 [] (len=1, tl=0)
	    @55989d5b1338 09 CHARSXP g0c1 [MARK,gp=0x60] [ASCII] [cached] ""
      ENCLOS:
	@55989d5e8498 04 ENVSXP g0c0 [MARK,NAM(2)] <R_EmptyEnv>
      ATTRIB:
	@55989d995468 02 LISTSXP g0c0 [] 
	  TAG: @55989d5b09f8 01 SYMSXP g0c0 [MARK,LCK,gp=0x4000] "class" (has value)
	  @55989e0e7d78 16 STRSXP g0c2 [NAM(2)] (len=2, tl=0)
	    @55989d65a210 09 CHARSXP g0c2 [MARK,gp=0x61] [ASCII] [cached] "srcfilecopy"
	    @55989d5b0ee8 09 CHARSXP g0c1 [MARK,gp=0x61] [ASCII] [cached] "srcfile"
      TAG: @55989d5b09f8 01 SYMSXP g0c0 [MARK,LCK,gp=0x4000] "class" (has value)
      @55989eba6ba8 16 STRSXP g0c1 [] (len=1, tl=0)
	@55989d5b0eb8 09 CHARSXP g0c1 [MARK,gp=0x61] [ASCII] [cached] "srcref"
```

When function `inspect` is called via `.Internal` what actually happens is that
the expression given as the argument to inspect is passed into a `SEXP`
structure and the C function `R_inspect(SEXP x)` defined in `inspect.c` is
called. 

This means that we can also run the inspect function from within C code (or
`gdb`) directly as `R_inspect(SEXP x)` and get the same information. The only
problem is that `R_inspect` is not visible outside of `inspect.c`, so you may
need to make it visible first.<sup>1</sup>

"OK," I hear you gasp in exasperation, "that's all fine and well, but what does
all this criptic garbage output actually mean?!" I'm gettin to that, keep your
pants on.

Getting to know them first
--------------------------

From the point of view of the internal C code of the R interpreter, SEXPs are a
pointer to a struct containing a header and a union payload, all defined in
`Rinternals.h`.  The header is defined like this:

```C
struct sxpinfo_struct {
    SEXPTYPE type      :  5; 
    unsigned int obj   :  1;
    unsigned int named :  2;
    unsigned int gp    : 16;
    unsigned int mark  :  1;
    unsigned int debug :  1;
    unsigned int trace :  1;
    unsigned int spare :  1;
    unsigned int gcgen :  1;
    unsigned int gccls :  3;
};
```

We'll ignore most of these, with the excetion of `type`. The `type` member
informs us what kind of SEXP we're dealing with and how to extract useful
information from its payload. This member is of type `SEXPTYPE`, which is
basically an `unsigned int`. For every type there is a corresponding constant
that's a little more hamster-readable. Here are the types and their constants
as defined in `Rinternals.h`:

```C
#define NILSXP	     0	  /* null */
#define SYMSXP	     1	  /* symbols */
#define LISTSXP	     2	  /* lists (specifically: pairlists) */
#define CLOSXP	     3	  /* closures */
#define ENVSXP	     4	  /* environments */
#define PROMSXP	     5	  /* promises */
#define LANGSXP	     6	  /* language constructs (special lists) */
#define SPECIALSXP   7	  /* special functions */
#define BUILTINSXP   8	  /* builtin non-special functions */
#define CHARSXP	     9	  /* internal string type*/
#define LGLSXP	    10	  /* logical vectors */
#define INTSXP	    13	  /* integer vectors */
#define REALSXP	    14	  /* real variables */
#define CPLXSXP	    15	  /* complex variables */
#define STRSXP	    16	  /* strings/character vectors */
#define DOTSXP	    17	  /* dot-dot-dot object */
#define ANYSXP	    18	  /* make "any" args work. */
#define VECSXP	    19	  /* generic vectors */
#define EXPRSXP	    20	  /* expression vectors */
#define BCODESXP    21    /* byte code */
#define EXTPTRSXP   22    /* external pointer */
#define WEAKREFSXP  23    /* weak reference */
#define RAWSXP      24    /* raw byte vector */
#define S4SXP       25    /* S4 classes, non-vector */

#define NEWSXP      30    /* fresh node created in new page */
#define FREESXP     31    /* node released by GC */
#define FUNSXP      99    /* closure or builtin or special */
```

You can ignore the last three and `ANYSXP`, since they are used as sort of
convenience markers within the interpreter. Eg. `FUNSXP` is used internally to
indicate either a SEXP of type `CLOSXP`, `BUILTINSXP`, or `SPECIALSXP`, but
should not actually occur as a type in ASTs created by parsing code; `ANYSXP`
is a placeholder value for any type of SEXP, etc. I'll discuss all the other
types below.

SEXPs have a number of convenience macros defined for them, so that it's not
usually necessary to actually retrieve information about them using the
definition of the struct. Thus, given a `SEXP` named `s` we can check its type
like this:

```C
if (TYPEOF(s) == NILSXP) {
    // ...
}
```

Apart from the header the SEXP contains the following union:

```C
union {
	struct primsxp_struct primsxp;
	struct symsxp_struct symsxp;
	struct listsxp_struct listsxp;
	struct envsxp_struct envsxp;
	struct closxp_struct closxp;
	struct promsxp_struct promsxp;
} u;
```

The component of the union on which to operate is chosen depending on the type:
`closxp` will have members that make sense for function definitions, `promsxp`
will have members represeting a promise, etc. There's rarely a need to look
into this union directly either, since there are ample macros that retrieve
particular members of particular SEXP types. I'll discuss them alongside
SEXP types below too.

Symbols
-------

SYMSXPs represent symbols, which are anything from operators, to names of
variables and function arguments. As such they also often appear as part of
other SEXPs.  Let's see one from the inside:

```R
x<-1
.Rinternal(inspect(substitute(x)))
```

```
@558fa07afe20 01 SYMSXP g0c0 [MARK] "x"
```

Inspect gives us the pointer address of this particular SEXP.  Upon further
inspection we also see this is a SEXP of type `01`, ie. indeed a `SYMSXP`. It
holds the value of `x` which represents the printable name of this symbol. In
order to extract that value in C we can use the `PRINTNAME` macro. This macro
returns another SEXP, a `CHARSXP`. `CHARSXP`s are the internal cacheable string
representation of R. You don't see them from R level, but you see them a lot of them 
when extracting information from other types of SEXPs. You can retrieve a
C-string from a `CHARSXP` with the `CHAR` macro. Example:

```C
const char *symbol_printname = CHAR(PRINTNAME(symbol));
```

A symbol may hold two additional pieces of information.

TODO what is SYMVALUE/SET_SYMVALUE. it can be R_UnboundValue
TODO what is INTERNAL it can be R_NilValue

Values and vectors
------------------

REALSXP
STRSXP
LGLSXP
CPLXSXP (complex(1,1)

Generic vectors
---------------

VECSXP   vector(mode="list", length=10), 
EXPRSXP  vector(mode="expression", length=10)

LISTSXP  vector(mode="pairlist", length=10) -- undocumented

TODO what's the difference between vecsxp and listsxp?

ASTs
----

LANGSXP

CLOSXP
------

Let's look at something more fun: closures. A closure is represented by a
CLOSXP, which contains three other SEXPs: its formals (argument definitions),
its body, and its environment.

RAWSXP
------

NILSXP
------

This is a singleton representing R's `NULL` value (`R_NilValue` from the point
of view of). There's nothing of interest inside.

```R
.Internal(inspect(substitute(NULL)))
```

```
@558fa0740d78 00 NILSXP g0c0 [MARK,NAM(2)] 
```











Footnotes
---------

1. For a quick an dirty hack see eg.: <https://github.com/PRL-PRG/R-dyntrace/commit/7083a563fc80262def3e33f7d7d1b8db903ca1c8#diff-b8536f70373bff169d16996af58b958b>

2. Actually, you don't need `substitute` for function definitions, you will get the same thing by running `.Internal(inspect(function(x)x))`.

References
----------

R Internals <https://cran.r-project.org/doc/manuals/r-release/R-ints.html>

