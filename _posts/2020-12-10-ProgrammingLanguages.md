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

# Registerization & Trampolining

Take the example of fib
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
2. Registerizing
  ```scheme
  (define fib-n #f)
  (define fib-k #f)
  
  (define fib
    (λ () #|n k|#
      (cond
        [(< fib-n 2)
         (let* ([k fib-k]
                [v 1])
           (apply-k k v))]
        [else
         (begin [set! fib-k (make-k-sub1 fib-n fib-k)]
                [set! fib-n (sub1 fib-n)]
                (fib))])))
  ```


A simple transformation of function calling

 when calling a func, make the args a simple var



 mk: rule -> prog

 - SPS
 - monadss