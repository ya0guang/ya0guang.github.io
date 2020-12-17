---
layout: single
title: Programming Language Review
date: 2020-12-10 13:10:07
categories: "Notes"
tags:
- Notes
- Course
- CS
- Theory
header:
  teaser: 
---

Review what I learned from programming language, a fundamental course of CS. The code is mainly in *typed* [racket](https://racket-lang.org/), a dialect of scheme. 

The code here is mainly from [Weixi Ma](https://github.com/mvcccccc)'s lecture notes and my assignments. The code from Weixi and me can be differentiated by indentation(2 spaces vs 4 spaces) and charset(unicode vs ascii).

# Useful resources

- [B521 Homepage](https://cgi.sice.indiana.edu/~c311/doku.php?id=home)

# Basics

## Natural Recursion

**Base case** + **recursion on a smaller part** of the input -> natural recursion

1. write down the boilerplate
  ```scheme
  (define +
  (λ (a b)
    (cond
      [... ...]
      [else ...])))
  ```
2. fill in the test for the base case
  ```shceme
  (define +
  (λ (a b)
    (cond
      [(zero? a) ...]
      [else ...])))
  ```
3. think hard about the result of the base case
  ```scheme
  (define +
  (λ (a b)
    (cond
      [(zero? a) b]
      [else ...])))
  ```
4. write down the natural recursion
  ```scheme
  (define +
  (λ (a b)
    (cond
      [(zero? a) b]
      [else ...(+ (sub1 a) b)...])))
  ```
5. make up examples that guide you from the natural recursive result to the real result
  > 3 plus 5 should be 8  
  > with our current recursion, (+ (sub1 3) 5) is 7  
  > then we should probably wrap the natural recursion with add1
6. write the wrapper
  ```scheme
  (define +
  (λ (a b)
    (cond
      [(zero? a) b]
      [else (add1 (+ (sub1 a) b))])))
  ```

## High order function

In functional programming, functions are [first-class citizen](https://en.wikipedia.org/wiki/First-class_citizen), which means parameter of a procedure that can itself be a procedure. Therefore we have high-order functions which takes a function as a parameter.

```scheme
(define map
  (λ (l f)
    (cond
      [(null? l) '()]
      [else (cons (f (car l)) (map (cdr l) f))])))
```

## Typed Racket

We are using [typed racket](https://docs.racket-lang.org/ts-guide/) this semester. It enforces type check when compared to the original racket.

```scheme
; Syntax
#lang typed/racket
(: functionName (-> para1Type para2ype ...
                    returnType))
(define (functionName para1 para2 ...)
  (...)

; some examples from AS1
; map
(: map (All (A B) (-> (-> A B) (Listof A) (Listof B))))
(define (map p ls)
    (cond 
        ((null? ls) ls)
        (else 
            (cons
                (p (car ls))
                (map p (cdr ls))))))

; append
(: append (All (A) (-> (Listof A) (Listof A) (Listof A))))
(define (append ls1 ls2)
    (cond 
        ((null? ls1) ls2)
        (else (cons (car ls1) (append (cdr ls1) ls2)))))

; append-map
(: append-map (All (A) (-> (-> A (Listof A)) (Listof A) (Listof A))))
(define (append-map p ls)
    (cond 
        ((null? ls) ls)
        (else 
            (append
                (p (car ls))
                (append-map p (cdr ls))))))
```

## Quotes

- **'**(quote): datum not including expression
- **`**(quasiquote): datum including expression
- **,**(unquote): for expression inside datum

## Lambda Calculus

[Lambda calculus](https://en.wikipedia.org/wiki/Lambda_calculus): a computation model, formal system in mathematical logic.

```scheme
(: λ? (→ Any Boolean))
(define λ?
  (λ (e)
    (match e
      [`,y
       #:when (symbol? e)
       #t]
      [`(λ (,x) ,body)
       (λ? body)]
      [`(,rator ,rand)
       (and (λ? rator) (λ? rand))]
      [`,whatever
       #f])))
```

## Free and bound

E.g.: `(λ (x) (x y))`
- occurs free: y
- occurs bound: x in (x y)

## Other snippets

```scheme
; examples from as2

; 4 walk-symbol 
(: walk-symbol (All (A B) (-> A (Listof (Pair A B)) (U A B))))
(define (walk-symbol x ls)
    (cond
        ((assv x ls) => (lambda (pr) (walk-symbol (cdr pr) ls)))
        (else x)))

(define-type Exp
  (U Symbol
     (List 'lambda (List Symbol) Exp)
     (List Exp Exp)))

; 13 lex
(define-type Exp-lex
  (U (List 'var Natural)
     (List 'lambda Exp-lex)
     (List Exp-lex Exp-lex)))

; helper function from a1
; list-index-ofv
(: list-index-ofv (All (A) (-> A (Listof A) Natural)))
(define (list-index-ofv s ls)
    (cond
        ((eqv? s (car ls)) 0)
        (else (add1 (list-index-ofv s (cdr ls))))))

(: lex (-> Exp (Listof Symbol) Exp-lex))
(define (lex e ls) 
    (match e
        [`,y #:when (symbol? y) 
            `(var ,(list-index-ofv y ls))]
        [`(lambda (,x) ,body) 
            `(lambda ,(lex body (cons x ls)))]
        [`(,rator ,rand) 
            `(,(lex rator ls) ,(lex rand ls))]))
```

# Interpreter and Environment

## Environment

Maps a symbol to it's meaning

```scheme
; e.g. Roman numbers
(define ROMAN-ENV
  (λ (y)
    (match y
      [`I 1]
      [`V 5]
      [`X 10]
      [`L 50]
      [`C 100]
      [`D 500]
      [`M 1000])))
```

## Basic interpreter <- λ-calculus

- var: look up in the env
- abs: make a closure, which wraps the env, body of the abs and the binding var
- app: get values of rator and rand and apply them

```scheme
(define-type Env
  (→ Symbol Meaning))
(define-type Closure
  (→ Meaning Meaning))
(define-type Meaning
  (U Number
     Closure))

(: valof (→ Exp Env
            Meaning))
(define valof
  (λ (exp env)
    (match exp
      [`,y
       #:when (symbol? y)
       (env y)]
      [`(λ (,x) ,body)
       (λ (arg) (valof body (λ (y) (if (eqv? y x) arg (env y)))))]
      [`(,rator ,rand)
       ((valof rator env) (valof rand env))])))
```

## Representation Independence

- high order func to represent env and closures
- data structures to represent them (lists)
- lambda can be eliminated in this way

```scheme
(: valof (→ Exp Env
            Meaning))
(define valof
  (λ (exp env)
    (match exp
      [`,y
       #:when (symbol? y)
       (apply-env env y)]
      [`,n
       #:when (number? n)
       n]
      [`(λ (,x) ,body)
       (make-closure x body env)]
      [`(pluz ,e₁ ,e₂)
       (match (valof e₁ env)
         [`,n₁
          #:when (number? n₁)
          (match (valof e₂ env)
            [`,n₂
             #:when (number? n₂)
             (+ n₁ n₂)]
            [`,whatever
             (error "not a number")])]
         [`,whatever
          (error "not a number")])]
      [`(,rator ,rand)
       (match (valof rator env)
         [`(closure ,x ,body ,env)
          (apply-closure `(closure ,x ,body ,env) (valof rand env))]
         [`,n
          #:when (number? n)
          (error "oops")])])))

(: apply-closure (→ Closure Meaning
                    Meaning))
(define apply-closure
  (λ (clos arg)
    (match clos
      [`(closure ,x ,body ,env)
       (valof body (ext-env x arg env))])
        ))
        
(: make-closure (→ Symbol Exp Env
                   Closure))
(define make-closure
  (λ (x body env)
    `(closure ,x ,body ,env)))

(: apply-env (→ Env Symbol
                Meaning))
(define apply-env
  (λ (env y)
    (match env
      ['()
       (error "oooooops, unound " y)]
      [`((,x . ,arg) . ,env)
       (if (eqv? y x) arg (apply-env env y))])))

(: ext-env (→ Symbol Meaning Env
              Env))
(define ext-env
  (λ (x arg env)
    `((,x . ,arg) . ,env)))

(: init-env (→ Env))
(define init-env
  (λ ()
    '()))
```

## De Bruijn Number

a.k.a. [De Bruijn index](https://en.wikipedia.org/wiki/De_Bruijn_index), to use numbers to represent the lexical address of a bound var in lambda-abstraction.

# Continuation Passing Style

A continuation is invoked at the end of function call and applied on the return value.

CPSing a function includes:
1. add a parameter k for continuation in def and every call to this func
2. in recursive call, create a continuation for it, which is applying continuation on the result of this step.
  ```scheme
  ; e.g.
  (f (g (h i) (j l)))
  ; ->
  (h i (lambda (hi) (f (g hi (j l)))))
  ```
3. do 2 to every non-trivial call!

```scheme
; e.g.: fib
(define fib
  (λ (n)
    (cond
      [(< n 2) 1]
      [else (+ (fib (sub1 n)) (fib (sub1 (sub1 n))))])))

(trace fib)
#;
(fib 5)

(define fib-cps
  (λ (n k)
    (cond
      [(< n 2) (k 1)]
      [else (fib-cps (sub1 n)
                     (λ (fib-sub-1)
                       (fib-cps (sub1 (sub1 n))
                                (λ (fib-sub-2)
                                  (k (+ fib-sub-1 fib-sub-2))))))])))
```
check [this link](https://github.com/ya0guang/ProgrammingLanguageCourse/blob/master/ContinuationPassingStyle/a4.rkt) for more examples!

## CPSed Interpreter

We can also CPS an interpreter!

```scheme
(define valof
  (λ (exp env)
    (match exp
      [`,y
       #:when (symbol? y)
       (env y)]
      [`,n
       #:when (number? n)
       n]
      [`(λ (,x) ,body)
       (λ (arg) (valof body (λ (y) (if (eqv? y x) arg (env y)))))]
      [`(let/cc ,cc ,body)
       (let/cc k (valof body (λ (y) (if (eqv? y cc) k (env y)))))]
      [`(throw ,k ,e)
       ((valof k env) (valof e env))]
      [`(pluz ,e₁ ,e₂)
       (+ (valof e₁ env) (valof e₂ env))]
      [`(,rator ,rand)
       ((valof rator env) (valof rand env))])))

(define valof-cps
  (λ (exp env-cps k)
    (match exp
      [`,y
       #:when (symbol? y)
       (env-cps y k)]
      [`,n
       #:when (number? n)
       (k n)]
      [`(λ (,x) ,body)
       (k (λ (arg k) (valof-cps body (λ (y k^) (if (eqv? y x) (k^ arg) (env-cps y k^))) k)))]
      [`(let/cc ,cc ,body)
       (valof-cps body (λ (y k^) (if (eqv? y cc) (k^ k) (env-cps y k^))) k)]
      [`(throw ,k ,e)
       (valof-cps k env-cps
                  (λ (k)
                    (valof-cps e env-cps k)))]
      [`(pluz ,e₁ ,e₂)
       (valof-cps e₁ env-cps
                  (λ (n₁)
                    (valof-cps e₂ env-cps
                               (λ (n₂)
                                 (k (+ n₁ n₂))))))]
      [`(,rator ,rand)
       (valof-cps rator env-cps
                  (λ (clos)
                    (valof-cps rand env-cps
                               (λ (arg)
                                 (clos arg k)))))])))
```

And also make it representation independent!

```scheme
(define valof-cps
  (λ (exp env-cps k)
    (match exp
      [`,y
       #:when (symbol? y)
       (env-cps y k)]
      [`,n
       #:when (number? n)
       (apply-k k n)]
      [`(λ (,x) ,body)
       (apply-k k (λ (arg k) (valof-cps body (λ (y k^) (if (eqv? y x) (k^ arg) (env-cps y k^))) k)))]
      [`(let/cc ,cc ,body)
       (valof-cps body (λ (y k^) (if (eqv? y cc) (k^ k) (env-cps y k^))) k)]
      [`(throw ,k ,e)
       (valof-cps k env-cps (make-k-throw e env-cps))]
      [`(pluz ,e₁ ,e₂)
       (valof-cps e₁ env-cps (make-k-pluz₁ e₂ env-cps k))]
      [`(,rator ,rand)
       (valof-cps rator env-cps (make-k-clos rand env-cps k))])))

(define apply-k
  (λ (k v)
    (k v)))

(define make-k-pluz₁
  (λ (e₂ env-cps k)
    (λ (v)
      (valof-cps e₂ env-cps
                 (make-k-pluz₂ v k)))))

(define make-k-pluz₂
  (λ (n₁ k)
    (λ (v)
      (k (+ n₁ v)))))

(define make-k-throw
  (λ (e env-cps)
    (λ (v)
      (valof-cps e env-cps v))))

(define make-k-clos
  (λ (rand env-cps k)
    (λ (v)
      (valof-cps rand env-cps
                 (make-k-arg v k)))))

(define make-k-arg 
  (λ (clos k)
    (λ (v)
      (clos v k))))

(define make-k-init
  (λ ()
    (λ (v) v)))
```

### η-reduction

```scheme
(lambda (a) (f a))
; can be simply reduced to 
f
```

# PC2C

A magic of this is to convert a racket program to a CPSed and registerized program which can be compiled into C language and an executable can then be generated.

More detailed instructions can be found [here](https://cgi.sice.indiana.edu/~c311/doku.php?id=assignment-9).

Take the example of fib:

```scheme
; original
(define fib
(λ (n k)
  (cond
    [(< n 2) (apply-k k 1)]
    [else (fib (sub1 n)
               (make-k-sub1 n k))])))

(define apply-k
  (λ (k v)
    (match k
      [`(init) v]
      [`(sub2 ,fib-sub1 ,k) (apply-k k (+ fib-sub1 v))]
      [`(sub1 ,n ,k) (fib (sub1 (sub1 n))
                          (make-k-sub2 v k))])))

(define make-k-sub1
  (λ (n k)
    `(sub1 ,n ,k)))

(define make-k-sub2
  (λ (fib-sub1 k)
    `(sub2 ,fib-sub1 ,k)))

(define make-k-init
  (λ ()
    `(init)))
```

1. Convert to [A-normal form](https://en.wikipedia.org/wiki/A-normal_form)
  ```scheme
  ; ANF
  (define fib
    (λ (n k)
      (cond
        [(< n 2)
        (let* ([k k]
                [v 1])
          (apply-k k v))]
        [else
        (let* ([k (make-k-sub1 n k)]
                [n (sub1 n)])
          (fib n k))])))
  ```
2. Registerizing: create global vars to hold the input parameters

  ```scheme
  (define fib-n #f)
  (define fib-k #f)
  (define apply-k-k #f)
  (define apply-k-v #f)

  (define fib
    (λ () #|n k|#
      (cond
        [(< fib-n 2)
        (begin [set! apply-k-k fib-k]
                [set! apply-k-v 1]
                (apply-k))]
        [else
        (begin [set! fib-k (make-k-sub1 fib-n fib-k)]
                [set! fib-n (sub1 fib-n)]
                (fib))])))

  (define apply-k
    (λ () #|k v|#
      (match apply-k-k
        [`(init) apply-k-v]
        [`(sub2 ,fib-sub1 ,k)
        (begin [set! apply-k-k k]
                [set! apply-k-v (+ fib-sub1 apply-k-v)]
                (apply-k))]
        [`(sub1 ,n ,k)
        (begin [set! fib-k (make-k-sub2 apply-k-v k)]
                [set! fib-n (sub1 (sub1 n))]
                (fib))])))
  
  (begin [set! fib-k (make-k-init)]
         [set! fib-n 5]
    (fib))

  ```
3. Add program counter and a list of continuations (RIP)

```scheme
  #lang racket
  (require "parenthec.rkt")

  (define fib-n #f)
  (define fib-k #f)
  (define apply-k-k #f)
  (define apply-k-v #f)

  (define-union continuations
    (init)
    (sub2 fib-sub1 k)
    (sub1 n k))
  ```

4. Change define to define-label and add `main`

  ```scheme
  #lang racket
  (require "parenthec.rkt")

  (define-program-counter pc)
  (define fib-n #f)
  (define fib-k #f)
  (define apply-k-k #f)
  (define apply-k-v #f)

  (define-union continuations
    (init jump-out)
    (sub2 fib-sub1 k)
    (sub1 n k))

  (define-label fib
    (cond
      [(< fib-n 2)
       (begin [set! apply-k-k fib-k]
              [set! apply-k-v 1]
              [set! pc apply-k])]
      [else
       (begin [set! fib-k (continuations_sub1 fib-n fib-k)]
              [set! fib-n (sub1 fib-n)]
              [set! pc fib])]))

  (define-label apply-k
    (union-case apply-k-k continuations
      [(init jump-out) (dismount-trampoline jump-out)]
      [(sub2 fib-sub1 k)
       (begin [set! apply-k-k k]
              [set! apply-k-v (+ fib-sub1 apply-k-v)]
              [set! pc apply-k])]
      [(sub1 n k)
       (begin [set! fib-k (continuations_sub2 apply-k-v k)]
              [set! fib-n (sub1 (sub1 n))]
              [set! pc fib])]))

  (define-label main
    (begin #;[set! fib-k (continuations_init)]
           [set! fib-n 5]
           [set! pc fib]
           [mount-trampoline continuations_init fib-k pc]
           (printf "~s\n" apply-k-v)))

  (main)
  ```

5. Delete racket stuffs which are already useless

# MiniKanren

MK~~小砍人~~ is a relational language. 

**Tip**: When it's hard to write down the program, try to think about its type first.

## APIs

- **==**: take 2 args, construct a goal which can be then ran to decide whether they can be euqal
- **run**: *(run n q g)* run the goal to determine *n* valuations *q* to satisfy the goal *g*
- **fresh**: create a set of fresh vars to be used in the body part
- **conde**: relational version of `cond`
- **defrel**: define a new relationship

## Implementation

### Basic Types

```scheme
; to denote variables and make it transparent(the content can be seen directly)
(struct Var
  ([name : Symbol])
  #:transparent)

(define-type Term
  (U Var
     Number
     Symbol
     Null
     (Pairof Term Term)))

; it's like an environment, but when looking up in a substitution, there will be no error (just return t if not found)
(define-type Substitution
  (Listof (Pairof Var Term)))
```

### Basic Operations

```scheme
; establish an association in a substitution if possible
(: unify (→ Term Term Substitution
            (U Substitution False)))
(define unify
  (λ (t₁ t₂ σ)
    (let ([t₁ (walk t₁ σ)]
          [t₂ (walk t₂ σ)])
      (cond
        [(eqv? t₁ t₂) σ]
        [(Var? t₁) (ext-s t₁ t₂ σ)]
        [(Var? t₂) (ext-s t₂ t₁ σ)]
        [(and (pair? t₁) (pair? t₂))
         (let ([σ^ (unify (car t₁) (car t₂) σ)])
           (and σ^ (unify (cdr t₁) (cdr t₂) σ^)))]
        [else #f]))))

; lookup in the substitution for the term (like in an environment)
(: walk (→ Term Substitution Term))
(define walk
  (λ (t σ)
    (cond
      [(Var? t)
       (cond
         [(assv t σ)
          =>
          (λ ([pr : (Pairof Var Term)])
            (walk (cdr pr) σ))]
         [else t])]
      [else t])))

(: occurs? (→ Var Term Substitution
              Boolean))
(define occurs?
  (λ (x t σ)
    (let ([t (walk t σ)])
      (cond
        [(Var? t) (eqv? t x)]
        [(pair? t) (or (occurs? x (car t) σ)
                       (occurs? x (cdr t) σ))]
        [else #f]))))

#|there shouldn't be a loop in a substitution|#
(: ext-s (→ Var Term Substitution
            (U Substitution False)))
(define ext-s
  (λ (x t σ)
    (cond
      [(occurs? x t σ) #f]
      [else `((,x . ,t) . ,σ)])))
```

### Look into MK's goal

```scheme
(define-type Goal
  (→ Substitution Stream))

(define-type Stream
  (U Null
     (Pairof Substitution Stream)
     (→ Stream)))

(: ≡ (→ Term Term Goal))
(define ≡
  (λ (t₁ t₂)
    (λ (σ)
      (let ([σ^ (unify t₁ t₂ σ)])
        (cond
          [σ^ `(,σ^)]
          [else '()])))))

(: take (→ Stream Number
           (Listof Substitution)))
(define take
  (λ ($ n)
    (cond
      [(zero? n) '()]
      [(null? $) '()]
      [(pair? $) (cons (car $) (take (cdr $) (sub1 n)))]
      [else (take ($) n)])))
```

### More complex stream opearations

```scheme
(: append$ (→ Stream Stream
              Stream))
(define append$
  (λ ($₁ $₂)
    (cond
      [(null? $₁) $₂]
      [(pair? $₁) (cons (car $₁) (append$ (cdr $₁) $₂))]
      [else (λ () (append$ $₂ ($₁)))])))

(: disj₂ (→ Goal Goal
            Goal))
(define disj₂
  (λ (g₁ g₂)
    (λ (σ)
      (append$ (g₁ σ) (g₂ σ)))))

(: append-map$ (→ Goal Stream
                  Stream))
(define append-map$
  (λ (g $)
    (cond
      [(null? $) '()]
      [(pair? $) (append$ (g (car $)) (append-map$ g (cdr $)))]
      [else (λ () (append-map$ g ($)))])))

(: conj₂ (→ Goal Goal
            Goal))
(define conj₂
  (λ (g₁ g₂)
    (λ (σ)
      (append-map$ g₂ (g₁ σ)))))
```

### A little bit on syntax rules

```scheme
(define-syntax andd
  (syntax-rules ()
    [(andd) #t]
    [(andd e0) e0]
    [(andd e1 e2) (if e1 e2 #f)]))

(define-syntax orr
  (syntax-rules ()
    [(orr) #f]
    [(orr e0) e0]
    [(orr e0 e1 ...) (let ([v e0])
                       (if v v (orr e1 ...)))]))
```

### Fresh and related marco

```scheme
(: call/fresh (→ Symbol (→ Var Goal)
                 Goal))
(define call/fresh
  (λ (x f)
    (f (Var 'x))))

(define-syntax fresh
  (syntax-rules ()
    [(fresh () g ...)
     (conj g ...)]
    [(fresh (x₀ x ...) g ...)
     (call/fresh 'x₀
                 (λ (x₀)
                   (fresh (x ...) g ...)))]))
              
(: reify (→ Term (→ Substitution Term)))
(define reify
  (λ (t)
    (λ (s)
      (let ([t (walk* t s)])
        (let ([r (reify-s t empty-s)])
          (walk* t r))))))

(: reify-s (→ Term Substitution
              Substitution))
(define reify-s
  (λ (t r)
    (let ([t (walk t r)])
      (cond
        [(Var? t) (let ([n (length r)])
                    (let ([name (string->symbol (string-append "_" (number->string n)))])
                      `((,t . ,name) . ,r)))]
        [(pair? t) (let ([r (reify-s (car t) r)])
                     (reify-s (cdr t) r))]
        [else r]))))
```

### Run in MK

```scheme
(define-syntax run
  (syntax-rules ()
    [(run n (x₀ x ...) g ...)
     (run n q
       (fresh (x₀ x ...)
         (≡ `(,x₀ ,x ...) q)
         g ...))]
    [(run n q g ...)
     (let ([q (Var 'q)])
       (map (reify q) (run-goal n (conj g ...))))]))

(: run-goal (→ Integer Goal
               (Listof Substitution)))
(define run-goal
  (λ (n g)
    (take$ n (g empty-s))))
```

# Type System

## Type built on MiniKanren

```scheme
; steal from lecture note
(defrel (∈ Γ x τ)
  (fresh (xa τa d)
    (== `((,xa . ,τa) . ,d) Γ)
    (conde
     [(== xa x) (== τa τ)]
     [(=/= xa x) (∈ d x τ)])))

; typed interpreter
(defrel (!- env exp t)
    (conde
        ; primary types
        [#| number |#
        (numbero exp)
        (== 'Nat t)]
        [#| true |#
        (== #t exp)
        (== 'Bool t)]
        [#| false |#
        (== #f exp)
        (== 'Bool t)]
        [#| var |#
        (symbolo exp)
        (∈ env exp t)]
        ; func with one arg
        [#| zero? |#
        (fresh (e1)
            (== `(zero? ,e1) exp)
            (== 'Bool t)
            (!- env e1 'Nat))]
        [#| sub1 |#
        (fresh (e1)
            (== `(sub1 ,e1) exp)
            (== 'Nat t)
            (!- env e1 'Nat))]
        [#| not |#
        (fresh (e1)
            (== `(not ,e1) exp)
            (== 'Bool t)
            (!- env e1 'Bool))]
        [#| car |#
        (fresh (ep td)
            (== `(car ,ep) exp)
            (!- env ep `(pairof ,t ,td)))]
        [#| cdr |#
        (fresh (ep ta)
            (== `(cdr ,ep) exp)
            (!- env ep `(pairof ,ta ,t)))]
        ; func with two args
        [#| plus |#
        (fresh (e1 e2)
            (== `(+ ,e1 ,e2) exp)
            (== 'Nat t)
            (!- env e1 'Nat)
            (!- env e2 'Nat))]
        [#| multi |#
        (fresh (e1 e2)
            (== `(* ,e1 ,e2) exp)
            (== 'Nat t)
            (!- env e1 'Nat)
            (!- env e2 'Nat))]
        [#| cons |#
        (fresh (e1 e2 t1 t2)
            (== `(cons ,e1 ,e2) exp)
            (== `(pairof ,t1 ,t2) t)
            (!- env e1 t1)
            (!- env e2 t2))]
        ; func with three args
        [#| if |#
        (fresh (etest ethen eelse)
            (== `(if ,etest ,ethen ,eelse) exp)
            (!- env etest 'Bool)
            (!- env ethen t)
            (!- env eelse t))]
        ; lambda calculas
        [#| abstraction |#
        (fresh (arg body ta tb)
            (== `(lambda (,arg) ,body) exp)
            (symbolo arg)
            (== `(,ta -> ,tb) t)
            (!- `((,arg . ,ta) . ,env) body tb))]
        [#| app |#
        (fresh (erator erand targ)
            (== `(,erator ,erand) exp)
            (!- env erator `(,targ -> ,t))
            (!- env erand targ))]
        ; recursion
        [#| fix |#
        (fresh (arg body)
            (== `(fix (lambda (,arg) ,body)) exp)
            (!- `((,arg . ,t) . ,env) body t))]
        ; dessert
        [#| let |#
        (fresh (arg sub body ta)
            (== `(let ([,arg ,sub]) ,body) exp)
            (symbolo arg)
            (!- `((,arg . ,ta) . ,env) body t)
            (!- env sub ta))]))
```

# Monads

Monad is like glue to things together in a pair. It can provide some convenient usages like one part can be used to represent the availability of the data and another to represent data itself. 

```scheme
; original fib
(define fib
  (λ (n)
    (cond
      [(zero? n) 0]
      [(zero? (sub1 n)) 1]
      [else (+ (fib (sub1 n))
               (fib (sub1 (sub1 n))))])))
```

## Monad Laws

- left-id: (bind (inj a) f) ≡ (f a)
- right-id: (bind ma inj) ≡ ma
- associative: (bind (bind ma f) g) ≡ (bind ma (λ (a) (bind (f a) g)))

## Memoization

> Use a hash table to avoid duplicate calculation if found in the table.

We store the data with in its return value and pass the memory as an argument to the next recursive call.

```scheme
#| memoization |#
(define fib/table
  (λ (n table)
    (cond
      [(hash-ref table n #f)
       =>
       (λ (result)
         `(,result . ,table))]
      [(zero? n) `(0 . ,table)]
      [(zero? (sub1 n)) `(1 . ,table)]
      [else (match-let* ([`(,fib-sub1 . ,table₁) (fib/table (sub1 n) table)]
                         [`(,fib-sub2 . ,table₂) (fib/table (sub1 (sub1 n)) table₁)]
                         [result (+ fib-sub1 fib-sub2)]
                         [table₃ (hash-set table₂ n result)])
              `(,result . ,table₃))])))

(define table '())
```

Or we may also store it somewhere else.

```scheme
(define table '())
(define fib/effect
  (λ (n)
    (cond
      [(assv n table)
       =>
       (λ (pr)
         (cdr pr))]
      [(zero? n) 0]
      [(zero? (sub1 n)) 1]
      [else (let* ([fib-sub₁ (fib/effect (sub1 n))]
                   [fib-sub₂ (fib/effect (sub1 (sub1 n)))]
                   [result (+ fib-sub₁ fib-sub₂)])
              (begin (set! table `((,n . ,result) . ,table))
                     result))])))
```

## State monad

State monads (State A S) package a value(A) and a state(effect part). It provide interfaces:
- injection  (put something at the value part) (→ A (State A S))
- bind (always bind a monads with a function that returns another monad) (→ (State A S) (→ A (State B S)) (State B S))
- get  (take the effect part)  : (→ (State A S) S)
- put  (put something at the effect part) (→ S (State A S))

Looking to the state monad:
```scheme
; create a new struct called `State` with two parameters `Store` and `A`. 
; State has one element called `run-state` which is a function that takes a `Store` and returns a pair of `A` and `Store`
(struct (Store A) State
  ([run-state : (-> Store (Pair A Store))]))

(: inj-state (All (S A) (-> A (State S A))))
(define (inj-state a)
  (State (λ ([s : S]) (cons a s))))

(: run-state (All (S A) (-> (State S A) (-> S (Pair A S)))))
(define run-state State-run-state)

(: bind-state (All (S A B) (-> (State S A)  (-> A (State S B)) (State S B))))
(define (bind-state ma f)
  (State (λ ([s : S])
           (match ((run-state ma) s)
             [`(,a . ,s) ((run-state (f a)) s)]))))

; get here is not returning a store?
(: get (All (S) (-> (State S S))))
(define (get)
  (State (λ ([s : S])
           (cons s s))))

(: put (All (S) (-> S (State S Unit))))
(define (put s)
  (State (λ (_) (cons (Unit) s))))
```


Now we can use state monad to rewrite fib.

```scheme
(define fib/state
  (λ (n)
    (bind-state (get)
                (λ (table)
                  (cond
                    [(assv n table)
                     =>
                     (λ (pr)
                       (inj-state (cdr pr)))]
                    [(zero? n) (inj-state 0)]
                    [(zero? (sub1 n)) (inj-state 1)]
                    [else
                     (bind-state (fib/state (sub1 n))
                                 (λ (fib-sub₁)
                                   (bind-state (fib/state (sub1 (sub1 n)))
                                               (λ (fib-sub₂)
                                                 (let ([result (+ fib-sub₁ fib-sub₂)])
                                                   (bind-state (get)
                                                               (λ (table)
                                                                 (bind-state (put `((,n . ,result) . ,table))
                                                                             (λ (_) (inj-state result))))))))))])))))
```

And use the marco `go-on` to make it easier:
```scheme
(define fib
  (λ (n)
    (go-on ([table (get)])
      (cond
        [(assv n table)
         =>
         (λ (pr)
           (inj-state (cdr pr)))]
        [(zero? n) (inj-state 0)]
        [(zero? (sub1 n)) (inj-state 1)]
        [else
         (go-on ([fib-sub₁ (fib (sub1 n))]
                 [fib-sub₂ (fib (sub1 (sub1 n)))])
           (let ([result (+ fib-sub₁ fib-sub₂)])
             (go-on ([table (get)]
                     [_ (put `((,n . ,result) . ,table))])
               (inj-state result))))]))))

#;
((run-state (fib 1000)) '())
```

## Maybe monad `(Maybe A)`

- injection or Just: (→ A (Maybe A))
- bind-maybe (→ (Maybe A) (→ A (Maybe B)) (Maybe B))
- Nothing puts something at the effect part (→ (Maybe A))

It can help people differentiate if there is something called `Nothing` or there is nothing.

```scheme
(define-type (Maybe A)
  (U (Just A)
     Nothing))
```

## `(Listof A)` is a monad

- bind  (→ (Listof A) (→ A (Listof B)) (Listof B)) is append-map
- inj (→ A (Listof A)) is (λ (a) (cons a '()))
- empty-list (→ (Listof A)) is (λ () '())

## Writer monad (log)

- bind (→ (Writer A Log) (→ A (Writer B Log)) (Writer B Log))
- inj  (→ A (Writer A Log))
- tell (→ Log (Writer A Log))

We can have a log of the execution trace.

```scheme
(define fib
  (λ (n)
    (cond
      [(zero? n)
       (go-on ([_ (tell `(fib ,n is 1))])
         (inj-writer 1))]
      [(zero? (sub1 n))
       (go-on ([_ (tell `(fib ,n is 1))])
         (inj-writer 1))]
      [else
       (go-on ([r₁ (fib (sub1 n))]
               [r₂ (fib (sub1 (sub1 n)))]
               [_ (tell `(fib ,n is ,(+ r₁ r₂)))])
         (inj-writer (+ r₁ r₂)))])))

(run-writer (fib 5))
```

## Continuation monad

```scheme
(define fib/k
  (λ (n)
    (cond
      [(zero? n)
       (inj-k 1)]
      [(zero? (sub1 n))
       (inj-k 1)]
      [else
       (go-on ([r₁ (fib/k (sub1 n))]
               [r₂ (fib/k (sub1 (sub1 n)))])
         (inj-k (+ r₁ r₂)))])))

((run-k
  (fib/k 5))
 (λ (v) v))
```

# Pie

It's a language for [dependent type](https://en.wikipedia.org/wiki/Dependent_type): a type whose definition depends on a value.

It introduces two operators: $\Pi$ and $\Sigma$, which are like *for any* and *there exists*.

We can claim these judgements (or say predicate?) in Pie

1. formation: _ is a type
2. construction: _ is _(some type)
3. sameness: _ is the _(some type) as _
4. _ is the same type as _

We can then have some kind of eliminators to iterate/recurse/induct on the elements of certain types.

```scheme
(claim apple Atom)
(define apple
  'apple)

(claim two-cats
  (Pair Atom Atom))
(define two-cats
  (car three-cats))

(check-same Atom 'cat (car two-cats))
```

## Nat

###  APIs

> The Nat type
> - formation rule: Nat is a type
> - constructors:
> 
> -------------------------
> zero : Nat
> 
> n : Nat
> --------------------------
> (add1 n) : Nat
> 
> - eliminators:
> 
> target : Nat
> base : X
> step : (→ Nat X)
> -------------------------------
> (which-Nat target base step) : X
> 
> -- (which-Nat zero base step) is the same X as base
> -- (which-Nat (add1 n) base step) is the same X as (step n)
> 
> -- (iter-Nat zero base step) is the same X as base
> -- (iter-Nat (add1 n) base step) is the same X as
> (step (iter-Nat n base step))
> 
> (rec-Nat target base step)
> 
> (rec-Nat zero base step) is the same X as base
> 
> (rec-Nat (add1 k) base step) is the same X as
> (step k (rec-Nat k base step))

### E.g.

```scheme
(claim sub1
  (→ Nat
    Nat))
(define sub1
  (λ (n)
    (which-Nat n
      0
      (λ (n-1) n-1))))

(claim *
  (→ Nat Nat
     Nat))
(define *
  (λ (n m)
    (iter-Nat n
      0
      (λ (almost)
        (+ m almost)))))

(claim fact
  (→ Nat
    Nat))
(define fact
  (λ (n)
    (rec-Nat n
      1
      (λ (k fact-of-k)
        (* (add1 k) fact-of-k)))))
```

## $\Pi$

>  (Π ([a A]) B)
> 1, formation
> A is a U
> B is a U if a is an A
> ----------------------------------
>  (Π ([a A]) B) is a U
> 
> 2, constructors
> 
> body is a B if x is an A 
> ---------------------------------
> (λ (x) body) is a (Π ([a A]) B)

## List

### APIs

> (List A)
> 1, formation:
>  (List A) is a type if A is a type
> 2, constructors:
> 
>  nil : (List A)
> 
>  a : A
>  l : (List A)
> ------------------------------
>  (∷ a l) : (List A)
> 
> 3, eliminators:
> (rec-List target base step)
> 
> (rec-List nil base step) is the same X as base
> 
> (rec-List (∷ a l) base step) is the same X as
> (step a l (rec-List l base step))
> 
> 4, sameness:
> 
> a₁ is the same A a₂
> l₁ is the same (List A) l₂
> ------------------------------------------------
> (∷ a₁ l₁) is the same (List A) as (∷ a₂ l₂)

### E.g.

```scheme
(claim length
  (Π ([A U])
    (→ (List A)
      Nat)))
(define length
  (λ (A)
    (λ (as)
      (rec-List as
        0
        (λ (a l length-of-l)
          (add1 length-of-l))))))
```

## Vector

### APIs

> (Vec A ℓ)
> 1, formation:
> 
> A : U
> ℓ : Nat
> --------------------
> (Vec A ℓ) : U
> 
> A is a type parameter
> ℓ is a type index
> E.g., (vec:: 'cat vecnil) (Vec Atom 0) -> (Vec Atom 1)
> 
> 
> 2, constructors:
> 
>  vecnil : (Vec A 0)
> 
> 
>  a : A
>  v : (Vec A ℓ)
> ------------------------------
>  (vec:: a v) : (Vec A (add1 ℓ))

## Induction

### `ind-Nat`
> (ind-Nat target motive base step)
> 
> 
>  motive : (→ Nat U)
>  base   : (motive 0)
>  step   : (Π ([k Nat] [almost (motive k)])
>             (motive (add1 k)))
> -------------------------------------------------------
> (ind-Nat target motive base step) : (motive target)

```scheme
(claim append
  (Π ([A U]
      [ℓ₁ Nat]
      [ℓ₂ Nat])
    (→ (Vec A ℓ₁) (Vec A ℓ₂)
       (Vec A (+ ℓ₁ ℓ₂)))))
(define append
  (λ (A ℓ₁ ℓ₂)
    (ind-Nat ℓ₁
      (λ (k)
        (→ (Vec A k) (Vec A ℓ₂)
           (Vec A (+ k ℓ₂))))
      (λ (xs ys) ys)
      (λ (k append-of-k)
        (λ (xs ys)
          (vec:: (head xs) (append-of-k (tail xs) ys)))))))
```

## Pie Implementation

We have to type of equations: $\alpha=$ and $\beta=$. $\beta$ type extend the representation of `Nat` in `add1`s and $\alpha$ type ignore the variable names (deBruijnize).

The basic idea of this part is to normalize and then de bruijinize the expressions and check if the result strings are equal. 
