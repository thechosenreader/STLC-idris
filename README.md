# An implementation of the STLC in Idris
An implementation of the simply typed lambda calculus (STLC) supporting pairs, atoms, lists, nats, and primitive recursion for naturals and lists.

Usually, an implementation of an STLC requires type-checking. This implementation leverages the dependent type system of the host language, Idris, to bypass the need for a type checker by making ill-typed programs impossible to represent. In effect, this delegates the process of type-checking to the host language (in this case, Idris). 

This delegation has some notable side effects. In particular, even though the STLC does not technically support polymorphic types, this implementation inherits them "for free" from the host language. 

Technically, the Idris code only implements an evaluator and normalizer. Since writing programs directly as syntax trees is cumbersome and error prone, we also provide a front-end language consisting of an S-expression based syntax which is translated into the Idris representation of an STLC program by the scripts: `stlc.py` and `run.sh`. For example, the command `./run.sh hello-world.rkt Output.idr` will take code written in this front-end syntax in the file `hello-world.rkt`, translate it into Idris and place the output in `Output.idr`, run the Idris compiler, and print the result. 

## A note on the host language
As mentioned above, this implementation is done in Idris. In Idris, types are "first-class" which enables an interplay between types and values that this implementation relies on to narrow the space of representable programs to exactly those which are well-typed.

More information about Idris can be found [on the official Idris website](https://www.idris-lang.org/).

## The code base
The evaluator and normalizer are written in Idris in `STLC.idr`. An overview of how it works is described below.

The file `sexpdata.py` contains an S-expression parser that was originally written by Takafumi Arakaki and Joshua Boyd and released under the BSD-2 license. Small modifications were made by the author to suit the purposes of this project. More information about this library can be found [in the original github repo](https://github.com/jd-boyd/sexpdata).

The above S-expression parser is used in `stlc.py` to define the syntax transformer for the front-end language (described in further detail below).

The bash script `run.sh` glues everything together; it takes as input the file containing the front-end source and (optionally) the output file. The name of the output file must adhere to the Idris language's rules on module file names; i.e, it must be capitalized and end in `.idr`. The script translates the front-end source into Idris code which it stores in the output file, runs the code and prints the result.

The `examples/` folder contains front-end source and the resulting Idris output. The naming convention is that front-end source-code is written in files with the `.rkt` extension, and the output of `<file>.rkt` is stored in `STLC_<file>.idr`.

## The front end language
Writing STLC programs directly with Idris constructors is cumbersome for a variety of reasons. To that end, the script `stlc.py` is provided to translate programs written using S-expression syntax into Idris code; the script `run.sh` then passes this translated code into the Idris compiler and prints the result.

The S-expression syntax itself has certain syntax sugar to support "multi-argument" functions. Thus, there are 3 syntaxes involved: 
 1. that of the host language Idris; programs are written directly as syntax trees composed of Idris data constructors.
 2. that of the "front-end" language; programs are written using S-expression syntax.
 3. that of the "front-end" language, plus some additional syntax sugar

In what follows, we describe the syntax of 2. and 3.
### Front-end syntax
The front-end language has `type`s, `expr`s, and two special forms `claim` and `define`. The syntax for types is:
```
<type> ::= Nat
         | Atom
         | (-> <type> <type>)
         | (Pair <type> <type>)
         | (List <type>)
```
Also, the syntax `(-> <type> <type> <type>*)` desugars to `(-> <type> (-> <type> (-> <type> <type>...)))` making it easier to represent functions of "more than one argument" (all functions are curried, so this is strictly a matter of convenient syntax).

Then, an `expr` is
```
<expr> ::= <identifier>
         | (lambda (<identifier>) <expr>)
         | (<expr> <expr>)
         | '<identifier>
         | zero
         | (add1 <expr>)
         | (rec-nat <expr> <expr> <expr>)
         | (cons <expr> <expr>)
         | (car <expr>)
         | (cdr <expr>)
         | nil
         | (:: <expr> <expr>)
         | (rec-list <expr> <expr> <expr>)
         | (the <type> <expr>)
```
Again, there is syntax sugar for defining multi-argument functions with `(lambda (<identifer> <identifier>*) <expr>)` and applying them with `(<expr> <expr> <expr>*)`.

There's also syntax sugar for defining nested pairs `(, <expr> <expr> <expr>*)` and lists `[<expr>*]`.

Finally, a name can be assigned a time with `claim`: `(claim <identifier> <type>)` and assigned an expression with `define`: `(define <identifier> <expr>)`.

Neither is strictly required for the other. A name can be claimed but not defined and defined but not claimed.

### Typing rules
This implementation follows standard typing rules for the STLC, as described below.

#### Lambda abstraction
If `e : b` in a context `C, x : a`, then `(lambda (x) e)` has type `(-> a b)` in context `C`.

#### Application
If `f : (-> a b)` and `x : a` in a context `C`, then `(f x)` has type `b` in context `C`.

#### Atom
In any context, for any identifer `x`, `'x` has type Atom.

#### Nat
In any context, `0 : Nat`. If `n : Nat` in context `C`, then `(add1 n) : Nat` in context `C`.

If, in context `C`, we have `n : Nat`, `b : a`, and `s : (-> Nat a a)`, then `(rec-nat n b s) : a` in the same context.

#### Pair
If `x : a` and `y : b` in context `C`, then `(cons x y) : (Pair a b)` in the same context.

If `p : (Pair a b)` in context `C`, then `(car p) : a` and `(cdr p) : b` in the same context.

#### List
In any context, `nil : (List a)`. If `x : a` and `l : (List a)` in some context `C`, then `(:: x l) : (List a)` in the same context.

If, in context `C`, we have `l : (List a)`, `b : c`, `s : (-> a (List a) c c)`, then `(rec-list l b s) : c` in the same context.

#### Annotation
If `x : a` in context `C`, then `(the a x) : a` in the same context. 

## Examples
The following are from the `examples/lib.rkt` file in this repository. Notice the lack of type annotations for `plus`, `prime?` and `nth-prime`: they are not needed, Idris is able to infer the type in each case. However, the annotation is required for `foldr`.

For each example, we also include the raw Idris code it translates to (after some manually done formatting).

### `plus`
```scheme
(define plus
    (lambda (m n)
        (rec-nat m n 
            (lambda (_ acc) (add1 acc)))))
```

```idris
plus : Expr ctx ?plus_t
plus = Lam 
        (Lam 
          (Rec 
            (Var (S Z)) 
            (Var Z) 
            (Lam 
              (Lam (Add1 (Var Z))))))
```
### `foldr`
```scheme
(claim foldr
    (-> (-> a b b) b (List a) b))
(define foldr
    (lambda (f i l)
        (rec-list
            l
            i
            (lambda (x _ acc)
                (f x acc))
            )))
```

```idris
foldr : Expr ctx $ (a :=> b :=> b) :=> b :=> (TList a) :=> b
foldr = Lam $
         Lam $
           Lam $
             RecList 
               (Var Z)
               (Var $ S Z) 
               (Lam 
                 (Lam 
                   (Lam 
                     (((Var $ S (S (S (S (S Z))))) :@ (Var (S (S Z)))) :@ (Var Z)))))
```

### `prime?` and `nth-prime`
In these examples, we make use of several functions which we have defined previously. The code for everything can be found in `examples/lib.rkt`.
```scheme
(define prime?
    (lambda (p)
        (== 2 (sum (lambda (x) (divides? x p)) p))))
```

```idris
primeq : Expr ctx $ ?primeq_t
primeq = Lam
          (eqNat :@ (Add1 $ Add1 Zero)
              :@ (
                sum :@ (Lam $ divides :@ (Var Z) :@ (Var (S Z)))
                    :@ (Var Z)
              )
          )
```

```scheme
(define nth-prime
    (lambda (n+1)
        (rec-nat
            n+1
            2
            (lambda (n pn)
                ; mu is bounded minimization
                ; will return the least natural satisfying the given predicate
                ; or the bound + 1 if there is none
                (mu (succ (! pn))
                    (lambda (z)
                        (* (prime? z) (> z pn))))))))
```

```idris
nth_prime : Expr ctx $ ?nth_prime_t
nth_prime = Lam 
  (Rec 
    (Var Z) 
    (Add1 $ Add1 Zero)
    (Lam 
      (Lam 
        (mu :@ (Add1 (factorial :@ (Var Z))) 
            :@ (Lam (times :@ (primeq :@ (Var Z)) 
                           :@ (gt :@ (Var Z) :@ (Var $ S Z))))))))
```

## How does it work?
### Overview
STLC programs are represented as values of an `Expr` type, indexed by a context and a type which respectively represent the types of the variables in scope and the type of the expression itself. 

Expressions evaluate to a value of a `Val` type, which is also indexed by a context and a type. The type still represents the type of the STLC value, but the role of the context is a little subtler. Essentially, the context is used to keep track of "neutral variables", which comes into play during normalization.

Values are "normalized" or "read back" into expressions which are their normal form. In this STLC, normal forms are unique, and so can be thought of as the simplest possible program which can produce a given value. Thus, `(lambda (x) (add1 (add1 x)))` and `(lambda (x) (add1 ((lambda (y) y) (add1 x))))` both "do the same thing", but the former is clearly "simpler".
### The `Ty` type
The `Ty` type represents the types of STLC values. We have naturals, atoms, functions, pairs, and lists, giving
```idris
data Ty
    = TNat
    | Atom
    | (:=>) Ty Ty
    | TPair Ty Ty
    | TList Ty
```
### Contexts and De Bruijn indices
Variables are represented by their De Bruijn index, which is a non-negative integer representing the number of binders between the variables occurrence and its initial binding site.

A context is simply a list of types. The type of the variable with De Bruijn index `i` is given by the `i`th element of the context. A variable of type `a` is constructed by providing a De Bruijn index for the given context.
### The `Expr` type
The `Expr` type is indexed by a context and a `Ty`. These two indices allow for the type of the constructors of the `Expr` type to fully encode the typing rules of the STLC. Consider:
```idris
data Expr : Context -> Ty -> Type where
    Var : DeBruijn ctx a -> Expr ctx a
    Lam : Expr (a :: ctx) b -> Expr ctx (a :=> b)
    (:@) : Expr ctx (a :=> b) -> Expr ctx a -> Expr ctx b
    Quote : String -> Expr ctx Atom
    Zero : Expr ctx TNat
    Add1 : Expr ctx TNat -> Expr ctx TNat
    Rec : Expr ctx TNat -> Expr ctx a -> Expr ctx (TNat :=> a :=> a) -> Expr ctx a
    Cons : Expr ctx a -> Expr ctx b -> Expr ctx (TPair a b)
    Car : Expr ctx (TPair a b) -> Expr ctx a
    Cdr : Expr ctx (TPair a b) -> Expr ctx b
    LNil : Expr ctx (TList a)
    LCons : Expr ctx a -> Expr ctx (TList a) -> Expr ctx (TList a)
    RecList : Expr ctx (TList a) -> Expr ctx x -> Expr ctx (a :=> (TList a) :=> x :=> x) -> Expr ctx x
    The : (a : Ty) -> Expr ctx a -> Expr ctx a
```
The type of the `Var` constructor ensures that one can't reference a variable which does not exist in the context. 

Similarly, the type of the `Lam` constructor can be read as the typing rule "if `f` has type `b` in a context with `x` a value of type `a`, then the the term `lambda x. f` has type `a :=> b`."
### The `Val` type
The `Val` type is defined by mutual recursion with the `Neutral` type and the `Env` type, which represent "neutral" STLC values and an evaluation environment.

A neutral value is a "stuck" computation; something that can not be evaluated further. This corresponds to elimination of values, so we have one constructor for each eliminator, as well as a constructor representing neutral variables:
```idris
data Neutral : Context -> Ty -> Type where
    NVar : DeBruijn ctx a -> Neutral ctx a
    NApp : Neutral ctx (a :=> b) -> Val ctx a -> Neutral ctx b
    NRec : Neutral ctx TNat -> Val ctx a -> Val ctx (TNat :=> a :=> a) -> Neutral ctx a

    NCar : Neutral ctx (TPair a b) -> Neutral ctx a
    NCdr : Neutral ctx (TPair a b) -> Neutral ctx b

    NRecList : Neutral ctx (TList a) -> Val ctx x -> Val ctx (a :=> (TList a) :=> x :=> x) -> Neutral ctx x
 ```

The `Env` type is indexed on _two_ contexts: the first is the context on which the values in the environment are all indexed, while the second gives the actual mapping between variables and their types:
```idris
data Env : Context -> Context -> Type where
    Nil : Env ctx' Nil
    (::) : Val ctx' a -> Env ctx' ctx -> Env ctx' (a :: ctx)
```

Finally, a `Val ctx a` corresponds to the introduction rules of the STLC, so we have one constructor for each way of introducing functions, nats, pairs, and lists:
```idris
data Val : Context -> Ty -> Type where
    VNeutral : Neutral ctx a -> Val ctx a
    VClosure : Env ctx' ctx -> Expr (a :: ctx) b -> Val ctx' (a :=> b)

    VZero : Val ctx TNat
    VAdd1 : Val ctx TNat -> Val ctx TNat

    VPair : Val ctx a -> Val ctx b -> Val ctx (TPair a b)

    VNil : Val ctx (TList a)
    VCons : Val ctx a -> Val ctx (TList a) -> Val ctx (TList a)

    VAtom : String -> Val ctx Atom
```
### Evaluation
Given an `Env ctx' ctx` and an `Expr ctx a`, we produce a `Val ctx' a` by the process of evaluation.
```idris
eval : Env ctx' ctx -> Expr ctx a -> Val ctx' a
```
Evaluating a variable corresponds to a lookup in the environment, which is guaranteed to succeed since variables can only be constructed via valid De Bruijn indices:
```idris
lookup : DeBruijn ctx a -> Env ctx' ctx -> Val ctx' a
lookup Stop (a :: r) = a
lookup (Pop k) (a :: r) = lookup k r

eval env (Var x) = lookup x env 
```
Generally, evaluation isn't too different from evaluation in non-dependently typed languages. The main difference is that the stronger types allow the compiler to verify for us that we aren't missing any cases. For example, as seen above, looking up a variable is guaranteed to succeed. Similarly, in evaluating a function application, the type index on the `Expr` and `Val` types is sufficient to prove that if we have an application of one term `f` to another, then `f` must evaluate to either a `VClosure` or a `VNeutral`.

So, we evaluate introduction rules (e.g `Zero`, `Add1`, `Cons`, etc.) by recursively evaluating the arguments and wrapping the result in the corresponding `Val` constructor. Likewise, we evaluate the elimination of neutral values by recursively evaluating the arguments and wrapping the result in the corresponding `Neutral` constructor.

Evaluating the elimination of non-neutral values dispatches on the exact eliminator: evaluating `rec-nat` and `rec-list` means carrying out a sequence of recursive evaluations which characterize the notion of primitive recursion embodied by these eliminators, and evaluating `car` and `cdr` mean simply choosing the first and second element of the pair.
### Normalization
In normalization, we want to take a `Val ctx a` and produce an `Expr ctx a`. Since a `Val` is either neutral or not, we split the normalization process into two functions:
```idris
readback : Val ctx a -> Expr ctx a

nereadback : Neutral ctx a -> Expr ctx a
```
The constructors of the `Neutral` type correspond directly to constructors of the `Expr` type, so normalizing them is fairly easy:
```idris
readback (VNeutral w) = nereadback w

nereadback : Neutral ctx a -> Expr ctx a
nereadback (NVar x) = Var x
nereadback (NApp f x) = (nereadback f) :@ (readback x)
nereadback (NRec n b s) = Rec (nereadback n) (readback b) (readback s)
nereadback (NCar p) = Car $ nereadback p
nereadback (NCdr p) = Cdr $ nereadback p
nereadback (NRecList xs b s) = RecList (nereadback xs) (readback b) (readback s)
```
Similarly, most of the `Val` constructors correspond directly to `Expr` constructors, so reading them back is easy:
```idris
readback VZero = Zero
readback (VAdd1 n) = Add1 $ readback n
readback (VPair a b) = Cons (readback a) (readback b)
readback VNil = LNil
readback (VCons x xs) = LCons (readback x) (readback xs)
readback (VAtom s) = Quote s
```
Normalizing a lambda abstraction is trickier, though, since the body of a lambda term might still contain unrealized computation that should be simplified. We can achieve this by applying a function value to a neutral variable, normalizing the result, and then wrapping that in `Lam`.

However, to apply a `Val ctx (a :=> b)`, we need to extend `ctx` with a value of type `a`. To do so, we define a relation `Contains` on contexts, whereby `Contains ctx ctx'` if and only if `ctx` can be obtained from `ctx'` by adding more types onto the front. The intuition here is that if `v` has type `a` in some context `C`, then adding more things into the context shouldn't affect the type of `v`.

In code, the relation `Contains` is defined as a data type:
```idris
data Contains : Context -> Context -> Type where
    Refl : ctx `Contains` ctx
    Weak : ctx `Contains` ctx' -> (a :: ctx) `Contains` ctx'
    Lift : ctx `Contains` ctx' -> (a :: ctx) `Contains` (a :: ctx')
```

Then, if `Contains ctx ctx'` and we have `v` a `Val ctx' a`, we should also be able to obtain a `Val ctx a`. Again, `Contains ctx ctx'` means `ctx` contains "more stuff" than `ctx'`, which shouldn't affect the type of `v`. So, we define
```idris
liftVal : ctx `Contains` ctx' -> Val ctx' a -> Val ctx a
```

In fact, since the `Val` type depends also on the `Env`, `Neutral`, and `DeBruijn` types, which all depend on a context, we also need to define
```idris
liftDeBruijn : ctx `Contains` ctx' -> DeBruijn ctx' a -> DeBruijn ctx a
liftEnv      : ctx `Contains` ctx' -> Env ctx' c -> Env ctx c
liftNeutral  : ctx `Contains` ctx' -> Neutral ctx' a -> Neutral ctx a
```

Finally, we can normalize a function value as follows:
```idris
readback v@(VClosure _ _) = Lam $ readback $ doApply (liftVal (Weak Refl) v) $ VNeutral (NVar Z)
```

