---
title: "Lambda"
date: 2022-10-27T22:13:14+08:00
draft: false
---

# Lambda演算

$\lambda$演算有三个最基本方面：

1. $\alpha$-变换
2. $\beta$-规约
3. 邱奇计数

## $\alpha$-变换

Alpha-变换规则陈述的是，若V与W均为变量，E是一个lambda表达式，同时E[V:=W]是指把表达式E中的所有的V的自由出现都替换为W，那么在W不是 E中的一个自由出现，且如果W替换了V，W不会被E中的λ绑定的情况下，有
$$
\lambda V.E == \lambda W.E[V:=W]
$$
所以本质上$\lambda x.(\lambda x) x$和$\lambda y.(\lambda x) y$是一致的。

## $\beta$-规约

$\beta$-规约陈述了若所有的E'的自由出现在E [V:=E']中仍然是自由的情况下，有
$$
((\lambda V.E) E') == E[V:=E']
$$
例子：
$$
(\lambda x. xy) z = zy 
$$

## 邱奇计数

邱奇计数中的定义：
$$
0 = \lambda f.\lambda x. x \\
1 =  \lambda f.\lambda x. f\ x \\ 
2 =  \lambda f.\lambda x. f (f\ x) \\
$$
依次类推，因此lambda演算中的数字n就是一个把函数f作为参数并以f的n次幂为返回值的函数。

因此类似可以定义后继函数，以n为参数，返回n+1：
$$
succ = \lambda n.\lambda f.\lambda x. f(n\ f\ x)
$$
同理定义加法、乘法：
$$
plus = \lambda m.\lambda n.\lambda f.\lambda x. f(n\ f\ x)
$$

$$
mult = \lambda m.\lambda n.m\ (plus\ n)\ 0
$$



一个racket实现，[源自](https://courses.cs.washington.edu/courses/csep505/13sp/lectures/church.rkt)：

```racket
#lang racket

;; Defines several concepts using only functions; namely:
;; - booleans and conditionals
;; - pairs
;; - natural numbers and arithmetic operators
;; - recursion
;;
;; Defines factorial in terms of these functions, along with a
;; way to convert from Church numerals to Racket numbers in order
;; to verify the result.

;; Church numeral representation of 0.
(define zero (λ (f) (λ (x) x)))

;; Returns the next Church numeral after n.
(define (succ n)
  (λ (f) (λ (x) (f ((n f) x)))))


;; Church numeral for 1.
(define one (succ zero))
(define two (succ one))
(define three (succ two))
(define four (succ three))

;; Adds two Church numerals.
(define (add n m)
  (λ (f) (λ (x) ((n f) ((m f) x)))))

;; Multiplies two Church numerals.
(define (mult n m)
  (λ (f)
    (n (m f))))

;; Computes n^m with Church numerals.
(define (exp n m)
  (m n))

;; Church encoding of true.
(define true (λ (x) (λ (y) x)))

;; Church encoding of false.
(define false zero)

;; Church encoding of the pair (x, y).
(define (make-pair x y)
  (λ (selector) ((selector x) y)))

;; Returns the first element of a pair.
(define (fst p)
  (p true))

;; Returns the second element of a pair.
(define (snd p)
  (p false))

;; Returns (x, x+1).
(define (self-and-succ x)
  (make-pair x (succ x)))

;; Given (x, y), returns (y, y+1).
(define (shift p)
  (self-and-succ (snd p)))

;; Returns the predecessor to the Church numeral n by
;; applying n to shift and (0, 0), then taking the first
;; element of the pair.
(define (pred n)
  (fst ((n shift) (make-pair zero zero))))

;; The eager Y combinator.
(define (fix f)
  ((λ (x) (λ (y) ((f (x x)) y)))
   (λ (x) (λ (y) ((f (x x)) y)))))

;; Returns whether a Church numeral is 0.
(define (is0 n)
  ((n (λ (x) false)) true))

;; An "if-then-else" function (the then and else branches
;; must be wrapped in zero-argument lambdas).
(define (ifte c t e)
  (((c t) e)))

;; The factorial function.
(define fact
  (fix (λ (f)
         (λ (n)
           (ifte (is0 n)
                 (λ () one)
                 (λ () (mult n (f (pred n)))))))))

;; Converts a Church numeral to a Racket number.
(define (as-number n)
  ((n add1) 0))

;; Converts a Church boolean to a Racket boolean.
(define (as-bool b)
  ((b #t) #f))
```
