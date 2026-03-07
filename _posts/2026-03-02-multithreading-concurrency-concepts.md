---
layout: post
title:  "Concurrency Concepts"
date:   2025-12-25 00:30:20 +0000
categories: programming python
---
## Concurrency
Its a way of fragmanting code so that individual fragmants can be run independently and reach the same result.  
E.g for taking average of x1,x2,x3....xn, we can do it by dividing all number in two segments like s1 = sum(x1, x2, x3....xm)/c1 and s2 = sum(xm1, xm2...xn)/c2 and then doing something like (s1+s2)/(c1+c2). This can be done by running fragmants on different cpu cores at the same or by sharing the same cpu (time-sliced execution).  

