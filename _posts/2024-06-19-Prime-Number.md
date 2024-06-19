---
layout: post
title:  "Prime Number"
date:   2024-06-19 15:01:27 +0800
categories: jekyll update
tags: 
    - Algorithm
---
# Intro
***Approach 1 gives the method to test whether n is a prime number.***

***Approach 2 gives the method to test whether numbers in a given range are prime numbers.***

# Approach 1
> *A prime number (or a prime) is a natural number greater than 1 that is not a product of two smaller natural numbers (except 1).*

- According to the definition, I iterate the ***pt*** from 2 to ***n*** to see if ***n*** is a prime number by testing ***pt*** is a factor of *n*.

- However, I could narrow the possible range of ***pt*** from 2 to ***n/2***, which the smallest possible factor of ***n*** is 2. For the same reason, 3 is the second smallest possible factor of ***n***.

- I could further simplify the range of ***pt*** because the biggest possible factor of ***n*** is ***sqrt(n)*** (a larger factor of n must be a multiple of a smaller factor that has been already checked).

# Code
```java
class Solution {
    // Approach 1: Simple way
    public boolean isPrime(int n) {
       if (n <= 1)
            return false;
 
        // Check from 2 to sqrt(n)
        for (int i = 2; i <= Math.sqrt(n); i++)
            if (n % i == 0)
                return false;
 
        return true;
    }   
}
```

# Approach 2
![Sieve_of_Eratosth](https://upload.wikimedia.org/wikipedia/commons/b/b9/Sieve_of_Eratosthenes_animation.gif)
> *Sieve of Eratosthenes is an ancient algorithm for finding all prime numbers up to any given limit.*

- First, assuming every 2k+1 number is prime number in the given range.

- Secondly, marking the frontest number which is currently prime (named i) as prime number and every i\*c (c is *N*) number as non-prime

- Third, simplifying the range of i\*c => i\*2c because every even c is non-prime. Then, start calculating from i\*i instead of i\*1 because i\*1 to i\*(i-1) is checked before.

# Code
```java
class Solution {
    // Appraoch 2: Sieve of Eratosthenes
    public boolean[] isPrime(int n) {
        if (n <= 2) return 0;
        boolean[] isPrime = new boolean[n];
        for (int i=3; i<n; i+=2) {
            isPrime[i] = true;
        }
        isPrime[2] = true;
        for (int i=3; i<Math.sqrt(n); i+=2) {
            if (isPrime[i]) { // if i is prime
                for (int j=i*i; j<n; j+=i*2) { // mark i*i, i*(i+2), i*(i+4),... as non-prime, which i=2k+1
                    isPrime[j] = false;
                }
            }
        }
        return isPrime;
    }
}
```

# Reference
- [Sieve of Eratosthenes](https://en.wikipedia.org/wiki/Sieve_of_Eratosthenes)
