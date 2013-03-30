# Intro to DP with example
(Excerpt from the [USACO training program](http://ace.delos.com/usacogate) text)

Dynamic programming is a confusing name for a programming technique that dramatically reduces the runtime of algorithms: from exponential to polynomial. The basic idea is to try to avoid solving the same problem or subproblem twice. 

### The problem

Here is a problem to demonstrate its power:

> Given a sequence of as many as 10,000 integers (0 < integer < 100,000), what is the maximum decreasing subsequence? Note that the subsequence does not have to be consecutive.

### Recursive Descent Solution (Naive)

The obvious approach to solving the problem is recursive descent. One need only find the recurrence and a terminal condition. Consider the following solution: 

```C
#include <stdio.h>
long n, sequence[10000];
main () {
    FILE *in, *out;                     
    int i;                             
    in = fopen ("input.txt", "r");     
    out = fopen ("output.txt", "w");   
    fscanf(in, "%ld", &n);             
    for (i = 0; i < n; i++) fscanf(in, "%ld", &sequence[i]);
    fprintf (out, "%d\n", check (0, 0, 999999));
    exit (0);
}
check (start, nmatches, smallest) {
    int better, i, best=nmatches;
    for (i = start; i < n; i++) {
        if (sequence[i] < smallest) {
            better = check (i, nmatches+1, sequence[i]);
            if (better > best) best = better;
        }
    }
    return best;
}
```

Lines 1-9 and and 11-12 are arguably boilerplate. They set up some standard variables and grab the input. The magic is in line 10 and the recursive routine `check`. The `check` routine knows where it should start searching for smaller integers, the length of the longest sequence so far, and the smallest integer so far. At the cost of an extra call, it terminates "automatically" when `start` is no longer within proper range. The `check` routine is simplicity itself. It traverses along the list looking for a smaller integer than the `smallest` so far. If found, `check` calls itself recursively to find more.

The problem with the recursive solution is the runtime: 

```
+----+---------+
| N  | Seconds |
+----+---------+
| 60 | 0.13    |
| 70 | 0.36    |
| 80 | 2.39    |
| 90 | 13.19   |
+----+---------+
```

Since the particular problem of interest suggests that the maximum length of the sequence might approach six digits, this solution is of limited interest.

### Starting At The End

When solutions don't work by approaching them "forwards" or "from the front", it is often fruitful to approach the problem backward. In this case, that means looking at the end of the sequence first.

Additionally, it is often fruitful to trade a bit of storage for execution efficiency. Another program might work from the end of the sequence, keeping track of the longest descending (sub-)sequence so far in an auxiliary variable.

Consider the sequence starting at the end, of length 1. Any sequence of length 1 meets all the criteria for a longest sequence. Notate the `bestsofar` array as `1` for this case.

Consider the last two elements of the sequence. If the penultimate number is larger than the last one, then the `bestsofar` is `2` (which is `1 + bestsofar` for the last number). Otherwise, it's `1`.

Consider any element prior to the last two. Any time it's larger than an element closer to the end, its `bestsofar` element becomes at least one larger than that of the smaller element that was found. Upon termination, the largest of the `bestsofar`s is the length of the longest descending subsequence.

This is fairly clearly an O(N^2) algorithm. Check out its code: 

```C
#include <stdio.h>
#define MAXN 10000
main () {
    long num[MAXN], bestsofar[MAXN];
    FILE *in, *out;
    long n, i, j, longest = 0;
    in = fopen ("input.txt", "r");
    out = fopen ("output.txt", "w");
    fscanf(in, "%ld", &n);
    for (i = 0; i < n; i++) fscanf(in, "%ld", &num[i]);
    bestsofar[n-1] = 1;
    for (i = n-1-1; i >= 0; i--) {
        bestsofar[i] = 1;
        for (j = i+1; j < n; j++) {
            if (num[j] < num[i] && bestsofar[j] >= bestsofar[i])  {
                bestsofar[i] = bestsofar[j] + 1;
                if (bestsofar[i] > longest) longest = bestsofar[i];
            }
        }
    }
    fprintf(out, "bestsofar is %d\n", longest);
    exit(0);
}
```

Again, lines 1-10 are boilerplate. Line 11 sets up the end condition. Lines 12-20 run the O(N 2) algorithm in a fairly straightforward way with the `i` loop counting backwards and the `j` loop counting forwards. One line longer then before, the runtime figures show better performance: 

```
+-------+---------+
|   N   | Seconds |
+-------+---------+
|  1000 | 0.08    |
|  2000 | 0.24    |
|  3000 | 0.55    |
|  4000 | 0.95    |
|  5000 | 1.45    |
|  6000 | 2.08    |
|  7000 | 2.99    |
|  8000 | 3.7     |
|  9000 | 4.7     |
| 10000 | 6.33    |
| 11000 | 7.35    |
+-------+---------+
```
The algorithm still runs too slow (for competitions) at N = 9000.

That inner loop (search for any smaller number) begs to have some storage traded for it.

A different set of values might best be stored in the auxiliary array. Implement an array `bestrun` whose index is the length of a long subsequence and whose value is the first (and, as it turns out, `best`) integer that heads that subsequence. Encountering an integer larger than one of the values in this array means that a new, longer sequence can potentially be created. The new integer might be a new "best head of sequence", or it might not. Consider this sequence: `10 8 9 4 6 3`.

Scanning from right to left (backward to front), the `bestrun` array has but a single element after encountering the 3: `0:3`

Continuing the scan, the `6` is larger than the `3`, to the `bestrun` array grows: 
```
0:3
1:6
```

The `4` is not larger than the `6`, though it is larger than the `3`, so the `bestrun` array changes: 
```
0:3
1:4
```

The `9` extends the array: 
```
0:3
1:4
2:9
```

The `8` changes the array similar to the earlier case with the `4`: 
```
0:3
1:4
2:8
```
The `10` extends the array again: 
```
0:3
1:4
2:8
3:10
```

and yields the answer: 4 (four elements in the array).

Because the `bestrun` array probably grows much less quickly than the length of the processed sequence, this algorithm probabalistically runs much faster than the previous one. In practice, the speedup is large. Here's a coding of this algorithm: 

```C
#include <stdio.h>
#define MAXN 200000
main () {
    FILE *in, *out;
    long num[MAXN], bestrun[MAXN];
    long n, i, j, highestrun = 0;
    in = fopen ("input.txt", "r");
    out = fopen ("output.txt", "w");
    fscanf(in, "%ld", &n);
    for (i = 0; i < n; i++) fscanf(in, "%ld", &num[i]);
    bestrun[0] = num[n-1];
    highestrun = 1;
    for (i = n-1-1; i >= 0; i--) {
    if (num[i] < bestrun[0]) {
          bestrun[0] = num[i];
          continue;
      }
      for (j = highestrun - 1; j >= 0; j--) {
          if (num[i] > bestrun[j]) {
              if (j == highestrun - 1 || num[i] < bestrun[j+1]){
                  bestrun[++j] = num[i];
                  if (j == highestrun) highestrun++;
                  break;
              }
          }
      }
    }
    printf("best is %d\n", highestrun);
    exit(0);
}
```

Again, lines 1-10 are boilerplate. Lines 11-12 are initialization. Lines 14-17 are an optimization for a new `smallest` element. They could have been moved after line 26. Mostly, these lines only effect the `worst` case of the algorithm when the input is sorted "badly".

Lines 18-26 are the meat that searches the bestrun list and contain all the exceptions and tricky cases (bigger than first element? insert in middle? extend the array?). You should try to code this right now - without memorizing it.

The speeds are impressive. The table below compares this algorithm with the previous one, showing this algorithm worked for N well into five digits: 

```
+-------+----------+----------+
|   N   | Original | Improved |
+-------+----------+----------+
|  1000 | 0.08     | 0.03     |
|  2000 | 0.24     | 0.03     |
|  3000 | 0.55     | 0.05     |
|  4000 | 0.95     | 0.06     |
|  5000 | 1.45     | 0.08     |
|  6000 | 2.08     | 0.09     |
|  7000 | 2.99     | 0.11     |
|  8000 | 3.7      | 0.13     |
|  9000 | 4.7      | 0.14     |
| 10000 | 6.33     | 0.16     |
| 11000 | 7.35     | 0.17     |
| 20000 |          | 0.29     |
| 40000 |          | 0.57     |
| 60000 |          | 0.91     |
| 80000 |          | 1.29     |
|100000 |          | 2.220    |
+-------+----------+----------+
```

### Summary

These programs demonstrate the main concept behind dynamic programming: build larger solutions based on previously found solutions. This building-up of solutions often yields programs that run very quickly.

For the previous programming challenge, the main subproblem was: Find the largest decreasing subsequence (and its first value) for numbers to the right of a given element.