
# Awelon

Awelon is a richly typed, compiled language for reactive demand programming (RDP). RDP is a general purpose, reactive dataflow model that addresses challenges involving open, distributed, heterogeneous systems (see AboutRDP for more). Awelon is intended to be a usable programming language for humans, a feasible distribution language for RDP code and software components, and a viable target for RDP development environments (some of which might be more visual in nature).

In Awelon, a single application can include code that executes across a distributed network. 

This Haskell project will provide an initial compiler for Awelon.

## Tacit Programming

Awelon code is [tacit](http://en.wikipedia.org/wiki/Tacit_programming). Juxtaposition is composition. All inputs are implicit, in an environment. However, unlike most tacit languages, Awelon's environment exists only as a compile-time type, and all manipulations to it are pure. Developers use the environment at a *conceptual* level - a way to organize resources and workflows, and to integrate them. 

At runtime, the environment is gone and the data plumbing is compiled completely away.

### Stack Based Programming

Like many tacit programming languages, Awelon supports stack-based programming. One of the key aspects of tacit programming is how *literals* are handled - e.g. if a developer writes `10 11 12`, what happens? Somehow, those literals must be added to the environment. For example, we can push literals onto a stack.

        42 %  e ~> (Static Integer * e)
        12.3 %  e ~> (Static Rational * e), exact 123/10
        "literal string" %  e ~> (Static Text * e)
        [swap] %  e ~> (Static ((a * b) ~> (b * a)) * e)

In Awelon, literals are simply numbers, text, or blocks of code. Developers often choose to distinguish rational numbers from integrals (e.g. by testing that the number is integral), but Awelon lumps them together. Also, due to polymorphism, Awelon essentially treats blocks as untyped until they're applied. Literals have 'Static' types, meaning their values are available at compile-time both for operations and decisions. Static types also are the key ingredient to make tacit programming work with an arrowized model, enabling us to model 'first' without use of explicit parameters.

        %  first :: (x ~> x') -> ((x*y) ~> (x'*y))              HASKELL
        %  first :: (Static (x ~> x') * (x * y)) ~> (x' * y)    AWELON
        [swap] first %  ((a*b)*c) ~> ((b*a)*c)

Support for partial application with 'first' is valuable for both code reuse and expressiveness. Awelon's model for 'first' has greater expressiveness because it allows the specification of the behavior to be cleanly separated from its call site, thus avoiding the need for an orthogonal abstraction model (such as lambdas). There is a price: Awelon's first is expressive enough to cause trouble. The traditional non-terminating lambda (`(\x->(x x) \x->(x x)`) can be represented (`[copy apply] [copy apply] first`).

To help illustrate what tacit code often looks like, I'll define a few more functions that assume a single stack:

        %  assocl :: (x * (y * z)) ~> ((x * y) * z)         PRIMITIVE
        %  swap   :: (x * y) ~> (y * x)                     PRIMITIVE
        %  intro1 :: x ~> (Unit * x)                        PRIMITIVE
        %  elim1  :: (Unit * x) ~> x                        PRIMITIVE
        %  copy   :: x ~> (x * x)                           polymorphic; see below
        
        apply  = [intro1 swap] second first swap elim1      %  (Static (x ~> x') * x) ~> x'
        second = swap [swap] first swap first swap          %  (Static (y~>y') * (x*y)) ~> (x*y')
        assocr = swap assocl swap assocl swap               %  ((x * y) * z) ~> (x * (y * z))
        roll2  = assocl [swap] first assocr                 %  (x * (y * z)) ~> (y * (x * z))
        roll3  = [roll2] second roll2                       %  (a * (b * (c * d)))) ~> (c * (a * (b * d)))
        roll4  = [roll3] second roll2
        dup    = [copy] first assocr                        %  (a * e) ~> (a * (a * e))
        pick1  = [dup] second roll2                         %  (a * (b * e)) ~> (b * (a * (b * e)))
        pick2  = [pick1] second roll2
        %  Note: the above definitions are for illustrations; those
        %  in use may be different (e.

Tacit code is easy to incrementally parse and process, and similarly easy to generate. Those are useful properties for Awelon's goals as a distribution or target language. Tacit code is naturally composable, and should be effective for representing software components. Avoiding direct manipulation by names enables many of RDP's invariants to be enforced constructively. Tacit code has very little syntactic noise. 

However, tacit code has a weakness: much of the data plumbing becomes explicit. Data plumbing words become a form of semantic noise, unrelated to the problem domain. For productive development, it is necessary that these data plumbing operations become simple, intuitive, fading into the background of the developer's attentions. 

*ASIDE:* Traditional languages also have difficulty with data plumbing - it just happens in higher layers: concurrency, callbacks, collections. RDP is designed to address many challenges in those higher layers.

### Awelon's Multi-Stack Environment

A single stack environment can feel cramped. This is especially the case for a model like RDP, where we often have multiple tasks or concurrent workflows that require ad-hoc integration of intermediate results. Awelon favors *multiple stacks for multiple tasks*, and thus provides an extended environment: 

        (currentStack * extendedEnvironment)

Stack programming operators like `roll` or `pick` are tweaked to operate on the current stack. Literals are now added to the current stack. 

        42                  %  (s*e) ~> ((N * s) * e) 
        12.3                %  (s*e) ~> ((N * s) * e) exact 123/10
        "literal string"    %  (s*e) ~> ((T * s) * e)
        [swap]              %  (s*e) ~> ((B * s) * e)

Other than that constraint, developers could create any extended environment they want. Adjusting the Awelon environment could be very useful to simplify integration with visual IDEs (i.e. such that gestures in the IDE correspond directly to streams of words). However, Awelon does provide a standard extended environment that should be expressive enough for human users:

        (currentStack * (hand * (currentStackName * listOfNamedStacks)))

The "hand" is another stack. Developers use `take` and `put` to move items between the hand and the current stack, and `juggle` to rotate items in the hand. The main purpose of the hand is to carry items between stacks, i.e. an intuitive approach to anonymous data plumbing. However, the hand is convenient even for operating on a single stack: one can pick up a few objects, do some deep-stack operations, then drop the objects again. This seems easier than carefully rolling objects to the top of the stack then back down.

Finally, there is a list of named stacks. Each element in the list is a (name*stack) pair, where the name is simply a text literal. Named stacks have a variety of purposes:

* Programmers can easily navigate stacks by name. This is useful for stacks that represent different tasks. The hand is convenient for carrying items between stacks. In a sense, this is a "goto", but it's more about the programmer's workflow than the program's.
* Programmers can `load` and `store`, obtaining values from a remote stack. This is useful for stacks that represent variables or registers, which is often convenient for long-term storage. 
* Named stacks can also support ad-hoc environment extensions, e.g. we could model aspect-oriented programming in Awelon in terms of explicit join-points (`"foo" signal`) and advice in the form of blocks. 

Named stacks require compile-time metaprogramming to systematically compare names in a list. Awelon's support for compile-time introspection and metaprogramming is discussed in a later section on polymorphic behaviors.
       

## Awelon's Module System

Awelon has a very simple module system. Essentially, a module consists of one import line followed by one or more definition lines. An import line contains a comma separated list of module names. The association between a module name and a module definition is externally determined (e.g. by file name). Module names must be simple words, i.e. valid as both Awelon export words and filesystem/URL components.

        import common, packageWithLongName AS p, foo
        this = [f] _s
        f = foo [p:bar foo:bar] first %  here 'foo' means 'foo:this'
        _s = swap [swap] first swap first swap

All words defined in a module are exported from it, except those prefixed with the underscore character such as _s above. (Imported words are not re-exported.) All words are also available prefixed with 'module:' from which they come. If there is any ambiguity for a word it becomes necessary to use the prefixed form. A word imported from multiple modules is not ambiguous if its multiple definitions are identical (same primitive expansion).

Words may be defined in any order within a module. Imports may be cyclic. However, *definitions* may not be cyclic or recursive, nor may they even appear to be. If developers wish to re-export a word, they must use the expanded form, e.g. `foo = foo:this` or `bar = p:bar`.

The word 'this' has special meaning. Within a module, 'this:' may be used as the prefix to disambiguate a module's words. When imported, the word 'this' is mapped instead to the import name, e.g. 'foo' can mean 'foo:this' after importing foo. Also, the word 'this' in Awelon serves the same role as 'main' in many other languages. Awelon is intended for component based software; I believe Awelon's use of 'this' has nice connotations to encourage treating modules as software components.

All imports and definitions must start at the beginning of a new line of text, and continue to the end of the logical line. A logical line continues until a non whitespace character begins a new line of text, i.e. so developers can distribute some code or imports vertically. Awelon has simple line comments starting eith `%`. 

        import common
        % takem :: (sL*((x*sC)*sR))*(hL*hR) ~> (sL*(sC*sR))*(hL*(x*hR))
        %   (older version of Awelon's environment; TODO, update for new env 
        takem= [[assocr] second roll2] first %  (x*(sL*(sC*sR)))*(hL*hR)
               assocr roll2 [roll2] second

There are no default words or imports in Awelon. Only literals can be used without an import. If there is something like a common prelude or standard environment, it must be on the import line. Consequently, it is not difficult for developers to create alternative programming environments (the only requirement is that literals are added to the first element of a pair).

*NOTE:* I believe developers shouldn't need more than a line of boiler plate before they get to useful work, which is why I allow multiple imports on one line. I discourage deep, hierarchical module systems. Libraries should generally be exported at the package layer. Ambiguity is rare in practice, and resolved easily enough. Modules as software components are encouraged to export 'this' and little else.

### Awelon Virtual Machine

The Awelon Virtual Machine (AVM) is a description of an RDP-based programming environment. 

This description includes types, partitions, and behavior primitives. It is a complete description, repeating even the ‘standard’ primitives like first and compose. Unlike a traditional VM, an AVM can describe heterogeneous systems, where different partitions have different behavior primitives and asymmetric communication. An AVM may additionally include: equational laws, proposed rewrite rules, and other properties. An application model would describe behavior types for applications or pluggable applets, along with some English text about how it is used. 

Within the avm, the type of a behavior is represented as a block to execute on a type descriptor. The 

The AVM is represented as an Awelon module that operates in a restricted subset of Awelon to construct its description (i.e. as opposed to using JSON or XML). This restricted subset is provided by a module called 'avmboot'. The AVM may import avmboot or other modules that ultimately depend on avmboot (rather than avmprim). The parser must understand avmboot implicitly during the bootstrap process. It provides just a few primitives for static operations.

Essentially, the AVM is a machine-processed standards document. There won't be many AVMs (probably the main line, a few experimental branches, maybe a didactic subset). After Awelon matures, the AVM will evolve very slowly. 

An Awelon application is limited to one AVM. Often, which AVM will be implicit in the development environment, and a compatible AVM can be switched in transparently. (An incompatible AVM can also be switched in; it just breaks your code.) In general, compilers will be limited to a single evolving line of AVMs.

### Primitive Words

A special module called 'avmprim' is created automatically from the AVM. This contains all the primitive words defined in the AVM, and is the ultimate root of all modules in an Awelon project. 

In general, developers do not directly import the primitives module. Primitives are too far removed from the more complicated multi-stack environments presented to developers. Even if they want low-level primitives, developers should use a module that extends them with all the common helpers (which I might name 'pure').

### Awelon Objects

Modules in Awelon are called Awelon Objects or AOs. The extension **.ao** should be used for modules represented in the filesystem (thogh I would like to move Awelon to a more wiki-like environment). AO files should be encoded in UTF-8. The name of the module is simply the filename excluding the extension. If there are ambiguous filenames (e.g. with installed packages), they must be resolved externally.

## Polymorphic Behaviors

In many cases, developers want a word to have a 'higher level' meaning whose precise application depends on context. For example, the word 'copy' should duplicate any element to which it is applied. Awelon has several behaviors that are polymorphic (first, swap, assocl, etc.) but none of them support 'copy' directly because not all types in Awelon can be copied (e.g. linear types cannot be copied). 

Awelon supports ad-hoc polymorphism by introspection of types and static values. 

For example, a 'copy' behavior might look at its argument and decide between a copyStaticText vs. copyStaticInteger primitive. Deep copies, on an `(x*(y*z))` structure or similar, can be achieved by structural induction by use of combinators.

### Static Choice

Awelon supports a concept of static choice `(x | y)` distinct from from dynamic sums `(x + y)`. A static choice is made based on observing a static value or introspecting a static type. A sum may be static or dynamic. Compilers can optimize a static sum under the hood, but developers must assume (with regards to type safety) that a sum is dynamic. Developers have much greater control and awareness of static choice.

Static choice is the basis for ad-hoc polymorphism. Developers can introspect types and values to make decisions:

        %  basic structure exposing primitives
        testIsProduct :: x ~> ((a * b) | x)
        testIsSum :: x ~> ((a + b) | x)
        testIsChoice :: x ~> ((a | b) | x)
        testIsOffer :: x ~> ((a & b) | x)
        testIsNumber :: x ~> (Static Rational | x) %  integer or rational
        testIsIntegral :: x ~> (Static Integer | x) %  just integers
        testIsText :: x ~> (Static Text | x)
        testIsBlock :: x ~> (Static (a ~> b) | x)
        testNumLessThan :: (Static Number * Static Number) ~> (Unit | Unit)
        testLexicalOrder :: (Static Text * Static Text) ~> (Unit | Unit)

More such operations may be available from the AVM. 

Note that programmers cannot test the type of a block. Programmers should *know* the type, much as they would in an untyped language. They can wrap a block with static hints and promises. But they can't actually test for type because blocks can also have ad-hoc polymorphism and there may not be a finite answer to the question, "what are your types?"

Basic operations one might expect for a choice are also supported:

        choiceFirst  :: (Static (x ~> x') * (x | y)) ~> (x' | y)
        choiceAssocl :: (x | (y | z)) ~> ((x | y) | z)
        choiceSwap   :: (x | y) ~> (y | x)
        choiceDist   :: (x * (y | z)) ~> ((x * y) | (x * z))
        choiceIntro0 :: x ~> (0 | x)
        choiceElim0  :: (0 | x) ~> x
        %  etc.

One of the big advantages of static choice is that all-powerful static disjoin primitive. The runtime version of that primitive is much more constrained.

Modeling choice in this first-class manner is much more extensible, expressive, and simpler than passing blocks as arguments. But eventually, developers will want to stop wrapping every operation up with 'choiceLeft'. Since developers are generally aware of which choice they're in, they're free to forget:

        choiceForget :: (x | y) ~> x OR  (x | y) ~> y %  choice no longer visible

After forgetting, developers should either be writing for the correct choice, or writing more polymorphic code that doesn't really care which choice was made. The runtime version of choice is 'merge' but is constrained to have identical types.

### Recursion and Repetition

A common situation, as with 'copy', is that developers need to operate on deep structures. For basic tree structures, we can take advantage that the type of an Awelon program is always finite. A simple recursion combinator for a tree might look like:

        mapTree :: OnLeaf * JoinTrees * Trees ~> whatever

Of course, we may want more complicated combinators: static folds, searches, etc.. Awelon doesn't build in any particular combinators, and instead leverages its compile time expressiveness. Awelon is Turing complete:

        (λx.(x x) λx.(x x)) %  "Forever" in Lambda Calculus.
        [copy apply] [copy apply] apply %  "Forever" in Awelon

Anyhow, Awelon must use a [fixpoint combinator](http://en.wikipedia.org/wiki/Fixed-point_combinator) and builds a library of recursive combinators atop it.

        %  a couple call-by-value fixpoint combinators from Curry and Turing
        Yv = λf. (λx.f (λv.((x x) v))) (λx.f (λv.((x x) v)))
        Θv = (λx. λy. (y (λz. x x y z))) (λx. λy. (y (λz. x x y z)))

At some point, I'll need to convert those into the associated Awelon behaviors. It can be done, I'm certain. But, right now, they make my head swim.

### Support for Sloppy Programming

There is a valuable role in this wide world for *sloppy* programming: rapid prototyping and executable pseudocode, plug-and-pray integration efforts, auto-configuration against available features, and live programming environments where active developers can easily gain feedback and refine on-the-fly.

Awelon's ad-hoc polymorphism should make a fair level of sloppiness feasible in terms of "smart" wiring logic and glue words, and leveraging latent choice extensively. The cost of sloppiness is a more expensive compile-time computation, since developers are pushing more work to the compiler.

### Aspect Oriented Awelon

A potential feature leveraging ad-hoc polymorphism is to model [aspect-oriented programming](http://en.wikipedia.org/wiki/Aspect-oriented_programming) in the Awelon compile-time. Basically, developers can model join-points using explicit `"foo" signal` code (to mark a "foo" join point). The 'signal' operation can look up the current advice (a block) associated with "foo" and effectively insert it in place of the signal. AOP can be very expressive.


## Awelon's Types

Awelon is very richly typed, though not in a conventional manner. Awelon types are computed - developers don't even have the option of declaring types. If developers wish to enforce specific types, they must use introspective compile-time behaviors. For static numbers, text, and blocks, the values are formally considered to be part of the type (which is useful to know for collections).

Awelon provides basic products, sums, statics, and collections. Awelon must also support a variety of runtime value types. And then there are a few more 'rich' types that make a huge difference in the programming experience and API design:

* fractional and negative types (right-to-left dataflow)
* linear types (enforcing contracts, protocol completion)
* uniquness source ('create' unique values or exclusive state)
* sealer/unsealer pairs (modeling encryption, ADTs)
* dynamic collections (sets, vectors)
* dynamic behaviors (runtime metaprogramming, linking)
* expiration and ripening (control latency)

### Fractional and Negative Types

To complete the set of algebraic types `(a * b)` and `(a + b)`, developers gain access to fractional and negative types: `1/a` and `-a`. These concepts are formally described in ["The Two Dualities of Computation: Negative and Fractional Types" by Amr Sabry and Roshan P James](http://www.cs.indiana.edu/~sabry/papers/rational.pdf). 

Fractional types are incredibly useful for modeling:

* adaptive code: polymorphic based on downstream requirements or preferences
* [promises, futures](http://en.wikipedia.org/wiki/Futures_and_promises), useful for quick integration without a lot of stack navigation since we can make promises in one place and fulfill them in another.
* a sort of 'continuation passing' along a pipeline 
* static negotiations between upstream and downstream components

I don't have an intuition or use-case for negative types. But they're symmetric and elegant, so they'll be supported too, and hopefully will find some use someday. 

Awelon introduces just a few primitive behaviors, plus the associated types:

        beat        :: 1 ~> (1/a * a)   
        backbeat    :: (1/a * a) ~> 1
        beat+       :: 0 ~> (-a + a)
        backbeat+   :: (-a + a) ~> 0

Here 'beat' takes the `1/a` as input, and outputs `a`. And 'backbeat' simply goes the other direction. (I getting some rock'n'roll when I named these, but I find them easy to remember.) Awelon is not doing anything clever here; e.g. there is no implicit conversion from `1/(1/a)` to `a`; fractional and negative types are essentially just new wrappers for old types.

A potential issue is that developers can now express invalid data-cycles:

               <-- 1/a --<       THIS IS BAD
              /           \
          beat             backbeat     
              \           /
               >--- a --->

Fortunately, these cycles are easy to detect statically. Unfortunately, these cycles are difficult to detect *compositionally*, i.e. in terms of shallow types without global analysis. The main consequence is that fractional types are not supported for dynamic behaviors; they can only be used internally to Awelon's static computation.

Use of beat and backbeat with block composition can also be used to twist inputs to different sides of blocks. The most useful operation is probably just to flip a block around and install it backwards. e.g. `(x~>y)~>(1/y~>1/x)`.

### Linear Types

Linear types are a very simple concept: values that cannot simply be dropped or copied, but must properly be consumed. These types are useful for enforcing that responsibilities are discharged, that protocols and handshakes are completed, that resources are used carefully, and so on.

In Awelon, linear types are simply modeled by NOT providing drop (`x ~> 1`) or copy (`x ~> x*x`) primitives for that type. (Awelon has an easier time of this than most languages - no worries about variables, closures, intermediate state, and so on.) Awelon can also model affine types (no copy) or relevant types (no drop). Many use-cases for linear types in Awelon involve static values. 

One simple linearity primitive is to mark blocks as linear:

        linearize :: (x~>y) ~> LinearBlock(x~>y)  

Linear blocks are consumed normally by first, left, etc.. They can also be composed, resulting in a larger linear block. But they cannot be dropped or copied. Basically, it's almost an invisible type tag. (*Note:* linearize actually involves marking a block both affine and relevant.)

In addition to enforcing properties, substructural types can help programmers organize their tasks - i.e. the type-system supports those "get back to this!" flags and "to do" lists.

### Uniqueness Source

The uniquness source is a static object that represents a source of "unique values". Developers can use this object for to create sealer/unsealer pairs or exclusive state resources. Due to the idempotence of RDP, Awelon primitive behaviors cannot 'create' a uniqueness source: if an application has uniqueness at all, the source must be an argument to the application. Most application models will provide uniqueness, though.

The uniqueness source can be dropped, but it cannot be copied in the traditional sense. Rather than copies, the source can be "forked", but each fork requires programmers provide a text identifier:

        fork :: (Text * U!) ~> (U! * U!')

The goal of this design is to ensure unique values are *stable* across changes in source code. The text provides a stable directory-like structure: we can think of 'fork' as a 'mkdir' operation in a filesystem, with the first output being the fresh directory and the second output being the modified parent directory that typefully forbids a second 'fork' with the same child name. Typically, the child (which has a fresh namespace) will be passed to a subprogram or otherwise consumed, while the parent will be stored away in the hand for later use. 

This resulting stability is valuable for live programming, debugging, runtime update, and orthogonal persistence. The text - forming a verifiably unique path in the code - is also usable in the implementation of unique values.


### Sealer/Unsealer Pairs

A sealer/unsealer pair can be created if you have a uniqueness source:

        newSealerUnsealerPair :: (Text * U!) ~> ((Sealer U! * Unsealer U!) * U!')
        type Sealer u = x ~> Sealed u x
        type Unsealer u = Sealed u x ~> x

Sealers and unsealers are just static blocks. They can be copied or dropped as normal. However, they are uniquely paired: the only means to unseal a value is to have a copy of the corresponding unsealer. The text associated with the sealer/unsealer pair is there mostly for debugging and error messages.

Sealer/unsealer pairs can be used in a number of creative ways. They can enforce parametricity or security properties. They can express rights-amplification patterns. They can model identity and blame (cf. [Horton's Who Done It](http://www.erights.org/elib/capability/horton/)). They can also model first-class abstract data types or existentials. 

In most use cases, sealer/unsealer pairs can be completely eliminated before runtime. However, in a distributed system, sealed values can guide encryption decisions for specific signals (orthogonal to transport-layer encryption). In particular, if sending information to a host that the compiler doesn't statically know will possess the unsealer, it may be worth encrypting the value. 

*Note:* Sealed values are implicitly linear. They must be unsealed before they can be dropped or copied.

### Abstract Data Types and Existentials

Awelon's introspection sometimes interferes with modularity, implementation-hiding, and user-defined types. However, idiomatic use of sealer/unsealer pairs can model a first-class module system, complete with ADTs or existentials. How this works:

1. The hidden type (an ad-hoc sum or product) is modeled by use of a sealer
2. A record of functions (product of blocks) is formulated to operate on the hidden type.
3. Each function is wrapped with the unseal/seal operations for the hidden type.
4. For ADTs, the record of functions is held separate from the sealed type
5. For existentials, the record of functions is coupled to each instance of the type

That's it. The sort of implementation hiding performed by most module systems is quite simple. Though, API designers must explicitly include the drop and copy methods if they don't want their new type to be linear.

In Awelon, these ADTs would implicitly be reactive, but would have relatively static structure.

### Dynamic Collections

Any general purpose programming language should have effective support for collections - lists or arrays - of both static and dynamic sizes. This is especially true for a reactive model, where there is a big difference between "a signal of arrays" (where the array updates atomically) and "an array of signals" (where each element may update independently). In Awelon, arrays and matrices of static size can be modeled by clever use of products and introspection; I'll leave those to libraries. 

However, primitive support is required for efficient, dynamic collections in Awelon.

Awelon will provide two dynamic collection types: vectors and sets of homogeneous type (which also means all static values must be the same). These collections will be processed by collection-oriented primitives, e.g. foreach operations, or folds for vectors. Vectors are the more expressive, having set operations plus support for order-dependent operations. 

*Desiderata:* supporting vectors with coupled dimensionality. Knowing two vectors have the same size, even if I don't know what is that size, would be very useful for some analyses. Also, I am hoping to avoid 'indexing' into vectors. 

### Dynamic Behaviors

Dynamic behaviors are, relatively, a second-class feature in Awelon and RDP. They operate under several constraints in order that they can be effectively implemented:

* cannot output static values (except unit)
* inputs are synchronized and start in one partition
* rigid structure for output types and latency
* no support for fractional or negative types

Despite these limitations, dynamic behaviors are useful for a wide variety of applications. They can model runtime resource or service discovery, plugins and extensions, authorities and capabilities, staged programming or metaprogramming, code distribution, or live update. Awelon application models will generally fit the dynamic behavior constraints.

I'm still deciding some aspects of this; dynamic behaviors might need some extra support to readily enforce static type safety. 


## Miscellaneous

What follows are ideas and potential idioms for Awelon users.

### Multi-Line Text

Awelon does not directly support multi-line text. However, developers can get the equivalent of multi-line text by using some sort of sentinel then using Awelon's static behaviors to construct the larger text:

        author = "William Wordsworth"
        daffo2 = linesStart
            "Continuous as the stars that shine"
            "And twinkle on the milky way,"
            "They stretched in never-ending line"
            "Along the margin of a bay:"
            "Ten thousand saw I at a glance,"
            "Tossing their heads in sprightly dance."
            linesEnd

Here, `linesStart` would place an obvious sentinel on the stack, and each line would then add to the stack. The `linesEnd` word would then introspect the stack, append the lines, and remove the sentinel, leaving the constructed text. This would occur at compile-time. (Developers aren't restricted to text literals, of course.)

### Block-Free Data Plumbing

Awelon's rich environment does have a cost: the primitive 'first' is no longer directly useful. Blocks are not placed where 'first' can apply them; further, most blocks would be designed to execute in the same complex environment. 

These issues are accommodated by developing a library of application behaviors that wrap a few values into a new environment, apply a block in the new environment, then unwrap and restore everything to their original locations. To simplify this effort, Awelon supports block-free data plumbing, such that blocks aren't added to the environment while trying to move objects around. 

The primitive data plumbing operators in Awelon are:

        assocl  :: (a * (b * c)) ~> ((a * b) * c)
        swap    :: (a * b) ~> (b * a)
        intro1  :: a ~> (Unit * a) 
        elim1   :: (Unit * a) ~> a
        rot3    :: (a * (b * (c * d))) ~> (c * (a * (b * d)))

The primitive rot3 was introduced just to enable block-free encodings. Together, these operators are very expressive. Any necessary pure data plumbing can be expressed without blocks. As a few examples:

        assocr = swap assocl swap assocl swap       % ((a * b) * c) ~> (a * (b * c))
        rot2 = intro1 rot3 intro1 rot3 elim1 elim1  % (a * (b * c)) ~> (b * (a * c))
        rot4 = assocl rot3 rot2 assocr rot3         % (a * (b * (c * (d * e)))) ~> (d * (a * (b * (c * e))))
        rot5 = assocl rot4 rot2 assocr rot3
        zip2 = assocr rot3 rot2 assocl              % ((a * b) * (c * d)) ~> ((a * c) * (b * d))

        % take :: ((x * s) * (h * ee)) ~> (s * ((x * h) * ee))
        % put  :: (s * ((x * h) * ee)) ~> ((x * s) * (h * ee))
        take = zip2 rot2
        put  = rot2 zip2
        
        % roll2 :: ((a * (b * s)) * e) ~> ((b * (a * s)) * e)
        % roll3 :: ((a * (b * (c * s))) * e) ~> ((c * (a * (b * s))) * e)
        roll2 = swap rot3 rot2 swap
        roll3 = swap rot4 rot2 swap
        roll4 = swap rot5 rot2 swap

Unfortunately, while very efficient, this approach to data plumbing isn't very intuitive. We can find patterns, easily guess what rot6 and roll5 might look like, but it isn't an implementation we'd provide inline if asked to think about it. A smarter approach is to rely on the structure of the environment:

        roll5 = take roll4 put roll2
        roll6 = take roll5 put roll2

Here, the intuiton is clearer. To roll the fifth item to the top: take one item off the top, roll what is now the fourth item to the top, put the item we took back on the stack, then roll it below what was once the earlier the fifth item. This is busier at compile-time, but the runtime overhead is still zero.

Other techniques are also available, for example based on [Huet's zippers](http://en.wikipedia.org/wiki/Zipper_(data_structure)), lenses, and traversals. Libraries of these data-plumbing behaviors will be developed.


### Stateful Programming

RDP requires that 

* long-running computations (e.g. ray-tracing, searching a filesystem)
* necessary state (e.g. text editors)

### Stateful Communication

There is always a temptation to use an external resource as a bypass, a way to communicate outside the lines and primary structure of the language. In most cases, giving into this temptation is a bad thing.

Awelon mitigates this temptation in two ways. First, Awelon supports ad-hoc data plumbing by use of hands or named stacks: when a signal is needed, a developer can get to it and take or copy the signal, often with less effort than it would take to set up shared state. Second, Awelon supports promises by use of fractional types: developers can "promise" a signal into existence then later fulfill it. 

However, stateful communication can be useful. And RDP is good at it.

Stateful communication is useful for:

* occasionally offline systems and disruption tolerance (like e-mail) 
* open systems where the audience is unknown (like a website) 
* open systems where the contributors are unknown (like a bulletin board)
* cases where we must time-share limited resources - e.g. we need task queues for printers
* cases where we don't know how long work will take - e.g. for ray-tracing or searching a filesystem

RDP is good at stateful communication because:

* vigilantly observe for changes in state, react immediately
* easily observe *multiple* states, react when all are appropriate
* easily support multiple observers, without update ordering issues
* support for speculative evaluation of states; anticipation of change
* external state enables orthogonal persistence by default

RDP was initially designed to support [publish-subscribe](http://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern) systems in a resilient, consistent, scalable manner. RDP is effective for integrating collaborative state: [blackboard system](http://en.wikipedia.org/wiki/Blackboard_system), [tuple spaces](http://en.wikipedia.org/wiki/Tuple_space), or even modeling ad-hoc [workflow patterns](http://en.wikipedia.org/wiki/Workflow_patterns).

Awelon further augments RDP by supporting exclusive state via the uniqueness source. Developers can enforce that certain state is one-to-one, one-to-many, or many-to-one. 

### Modeling Events

One of the most awkward problems to express in RDP is the concept of 'instantaneous event'. 

Fortunately, nothing we can observe is truly instantaneous. Good arguments can be made that even 'button presses' and similar examples should be modeled with time-varying signals. For example, if we use time-varying signals, we can easily recognize that alt+enter are pressed at the same time, without using intermediate state to model the world, and thus without risk of inconsistent state. Also, RDP state resources have no difficulty integrating short-lived signals. So real-world control systems can usually do without events.

But occasionally, someone will come up with a problem like: "Hey, I just shot you in this game. I rolled the dice and you should take 12 damage. How do I model this in RDP?"

The video game is its own reality, with its own laws, in this case including "I can tell you how much damage you took" events. We could change the rules, make the video-game more 'realistic' but that isn't the desired solution. Fortunately, despite being awkward, we can accommodate events. We use state

A 'take 12 damage' event must be applied only once, which means we must record that the event was applied, which means we need a unique identifier for 'the event' - e.g. if the event contains a UID and timestamp.

Use two states: a local state that records whether the event has been acknowledged (or at least delivered), and a remote state that receives the event. 

Given an event, we can continuously watch our local state to see whether we should deliver it. If so, we send the signal then (with a little delay) record that we've sent the event into the local state. 

The remaining challenge is to ensure each event is unique, e.g. by including a UID and time-value in the event, or by 

 acknowledgement just a little, then release the event to execute. 

One solution is to adjust the rules, make this video-game reality more like our own. 



Button presses, for example, are often better modeled as time-varying button state (especially if we want to recognize when multiple buttons are pressed simultaneously). Short-lived signals can easily be folded into state.

But occasionally, someone will come up with a problem like:  This is difficult to translate to RDP because we aren't working in reality anymore. We're working in a simulation, with events. 


#### Demand Monitors



#### Signal Sinks

A trivial but useful form of single-writer state is to have a 'stateful signal' that is continuously updated by inputs to it, but that holds its old value (along with the time of last synchronization) after disruption (perhaps with an expiration). This concept is used to send Awelon applications for remote, disruption-tolerant execution.

### Stable Stateless Models

### Exponential Decay State

### Agent-Resources


### Debugging

Errors, Warnings, Traces


### IDE Integration

translating drag and drop to words
automatic rewriting to simplify the program

Rendering Hints (relative positions?)

Gauges (some sort of: "if the user is near, install this here" or "when there's someone in this forest, here's the sound to make")

Diagrams-as-Data (idea: like Pure Data support diagrams as a form of input data, representing diagrams in type-system with static values).

Gauges 

Gauges? Maybe some way to say "hey, display this using XYZ, but only if we're looking at it"

Due to its support for static types, Awelon code can readily be augmented with 'active' documentation - e.g. commands that don't do anything except indicate how to display or debug the code itself. 







