# Burrows-Wheeler transform {#bwt}



The Burrows-Wheeler transform is one of the most effective lossless text compression method available. It provides a reversible transformation for text that makes it easier to compress. Of course, one may wonder what text compression has to do with genome assembly. As a matter of fact these two issues are closely related. But we should to be more precise here - text compression is closely related to pattern matching which in turn is crucial for the genome assembly. In a broad sense compression algorithms look for patterns and try to remove repetitions. We want to take advantage of this feature, especially because repetitive patterns tend to be very abundant in genomic sequences.

It is worth mentioning that the Burrows-Wheeler transform is also closely related to suffix trees and suffix arrays, which are commonly used within pattern matching. This relationship will be studied later but perhaps reader should already keep the trivia in mind. [@bw1]

## Introduction

The Burrows-Wheeler transform method is often referred to as “block sorting”, because it takes a block of text and permutes it. By permuting a block of text we mean rearranging the order of its symbols. Once again, we should be more precise here because Burrows-Wheeler transform performes a specific type of permutation, namely *circural shift permutation*: all of the characters are moved one position to the left, and first character moves to the last position.

## Burrows-Wheeler matrix

Consider the following sequence:

```{.r .numberLines}
sequence <- 'GATTACA'
```

In order to create the Burrows-Wheeler matrix, from which the transform itself can be obtained, for the given string we at first add the dollar sign $ at the end of the sequence.


```{.r .numberLines}
sequence  <- str_c(sequence, '$')
```

Afterwards we perform a series of circular shift permutations.


```{.r .numberLines}
sequences <- c(sequence)
n         <- nchar(sequence)

for (i in 1:(n-1)){
  sequence <- str_c(str_sub(sequence, 2, n),
                    str_sub(sequence, 1, 1))
  
  sequences <- c(sequences, sequence)
}

cat(sequences, sep = '\n')
#> GATTACA$
#> ATTACA$G
#> TTACA$GA
#> TACA$GAT
#> ACA$GATT
#> CA$GATTA
#> A$GATTAC
#> $GATTACA
```

Then we sort these sequences with the assumption that the dollar sign precedes lexicographically every other symbol.


```{.r .numberLines}
sequences <- sort(sequences) 
cat(sequences, sep = '\n')
#> $GATTACA
#> A$GATTAC
#> ACA$GATT
#> ATTACA$G
#> CA$GATTA
#> GATTACA$
#> TACA$GAT
#> TTACA$GA
```

For our convenience let's split these permutations into vectors of single characters.


```{.r .numberLines}
bw.matrix           <- data.frame(matrix(, n, n))
colnames(bw.matrix) <- 1:n

for (i in 1:n){
  bw.matrix[i, ] <- strsplit(sequences[i], split = '')[[1]]
}

knitr::kable(bw.matrix)
```



|1  |2  |3  |4  |5  |6  |7  |8  |
|:--|:--|:--|:--|:--|:--|:--|:--|
|$  |G  |A  |T  |T  |A  |C  |A  |
|A  |$  |G  |A  |T  |T  |A  |C  |
|A  |C  |A  |$  |G  |A  |T  |T  |
|A  |T  |T  |A  |C  |A  |$  |G  |
|C  |A  |$  |G  |A  |T  |T  |A  |
|G  |A  |T  |T  |A  |C  |A  |$  |
|T  |A  |C  |A  |$  |G  |A  |T  |
|T  |T  |A  |C  |A  |$  |G  |A  |

Thus we have created the **Burrows-Wheeler matrix**.  Sequence in the last column is called the **Burrows-Wheeler transform**.


```{.r .numberLines}
transform <- paste(bw.matrix[,n], collapse = '')

cat('The Burrows-Wheeler transform of', 
    sequence, 'is', transform)
#> The Burrows-Wheeler transform of $GATTACA is ACTGA$TA
```

## Inverse transform

As we said at the very beginning the transform is reversible. Having only the transformed sequence we are going to reconstruct the Burrows-Wheeler matrix and initial sequence itself. 

Firstly let's sort the characters of the transformed sequence.


```{.r .numberLines}
first.sequence <- strsplit(transform, split = '')[[1]] %>% sort
paste(first.sequence, collapse = '')
#> [1] "$AAACGTT"
```

Note that this string is equivalent to the first column of the Burrrows-Wheeler transform.


```{.r .numberLines}
bw.inverse           <- data.frame(matrix(, n, 2))
colnames(bw.inverse) <- c(n, 1)

bw.inverse[, 1] <- strsplit(transform, split = '')[[1]]
bw.inverse[ ,2] <- first.sequence

knitr::kable(bw.inverse)
```



|8  |1  |
|:--|:--|
|A  |$  |
|C  |A  |
|T  |A  |
|G  |A  |
|A  |C  |
|$  |G  |
|T  |T  |
|A  |T  |

Also keep in mind that the characters from last and the first column are adjacent. In other words, at this point we have a set of 2-mers.


```{.r .numberLines}
kmers <- apply(bw.inverse, 1, 
               function(x) paste(x, collapse = ''))
kmers
#> [1] "A$" "CA" "TA" "GA" "AC" "$G" "TT" "AT"
```

The reconstruction process strictly relies on the fact that Burrows-Wheeler matrix is sorted lexicographically. This property will allow us to retrieve the remaining columns.


```{.r .numberLines}
kmers <- sort(kmers)
kmers
#> [1] "$G" "A$" "AC" "AT" "CA" "GA" "TA" "TT"
```

The 2-mers (k-mers in general) that we sorted lexicographically represent first two columns of the Burrows-Wheeler matrix. We can extract last character of each 2-mer in the following way:


```{.r .numberLines}
sapply(kmers, function(x) str_sub(x, 2, 2), 
       simplify = TRUE, USE.NAMES = FALSE)
#> [1] "G" "$" "C" "T" "A" "A" "A" "T"
```

By inserting this set of characters we obtained the second column, and by iterating the proccess of building substrings, sorting them, and retrieving last characters we can fill the whole Burrows-Wheeler matrix.


```{.r .numberLines}
for (i in 2:(n-1)){
  kmers             <- apply(bw.inverse, 1, 
                             function(x) paste(x, collapse = ''))
  kmers             <- sort(kmers)
  bw.inverse[, i+1] <- sapply(kmers, function(x) str_sub(x, i, i), 
                              simplify = TRUE, USE.NAMES = FALSE)
  colnames(bw.inverse)[i+1] = i
}
knitr::kable(bw.inverse)
```



|8  |1  |2  |3  |4  |5  |6  |7  |
|:--|:--|:--|:--|:--|:--|:--|:--|
|A  |$  |G  |A  |T  |T  |A  |C  |
|C  |A  |$  |G  |A  |T  |T  |A  |
|T  |A  |C  |A  |$  |G  |A  |T  |
|G  |A  |T  |T  |A  |C  |A  |$  |
|A  |C  |A  |$  |G  |A  |T  |T  |
|$  |G  |A  |T  |T  |A  |C  |A  |
|T  |T  |A  |C  |A  |$  |G  |A  |
|A  |T  |T  |A  |C  |A  |$  |G  |

Finally we move first column to the very end


```{.r .numberLines}
bw.inverse[,n+1] <- bw.inverse[, 1]
bw.inverse       <- bw.inverse[,2:(n+1)]
colnames(bw.inverse)[n] = n

knitr::kable(bw.inverse)
```



|1  |2  |3  |4  |5  |6  |7  |8  |
|:--|:--|:--|:--|:--|:--|:--|:--|
|$  |G  |A  |T  |T  |A  |C  |A  |
|A  |$  |G  |A  |T  |T  |A  |C  |
|A  |C  |A  |$  |G  |A  |T  |T  |
|A  |T  |T  |A  |C  |A  |$  |G  |
|C  |A  |$  |G  |A  |T  |T  |A  |
|G  |A  |T  |T  |A  |C  |A  |$  |
|T  |A  |C  |A  |$  |G  |A  |T  |
|T  |T  |A  |C  |A  |$  |G  |A  |

One can also verify that bw.matrix and bw.inverse are in fact the same. 


```{.r .numberLines}
knitr::kable(bw.inverse == bw.matrix)
```



|1    |2    |3    |4    |5    |6    |7    |8    |
|:----|:----|:----|:----|:----|:----|:----|:----|
|TRUE |TRUE |TRUE |TRUE |TRUE |TRUE |TRUE |TRUE |
|TRUE |TRUE |TRUE |TRUE |TRUE |TRUE |TRUE |TRUE |
|TRUE |TRUE |TRUE |TRUE |TRUE |TRUE |TRUE |TRUE |
|TRUE |TRUE |TRUE |TRUE |TRUE |TRUE |TRUE |TRUE |
|TRUE |TRUE |TRUE |TRUE |TRUE |TRUE |TRUE |TRUE |
|TRUE |TRUE |TRUE |TRUE |TRUE |TRUE |TRUE |TRUE |
|TRUE |TRUE |TRUE |TRUE |TRUE |TRUE |TRUE |TRUE |
|TRUE |TRUE |TRUE |TRUE |TRUE |TRUE |TRUE |TRUE |

Additionally we can encapsulate the Burrows-Wheeler transform in a form of a single function.


```{.r .numberLines}
BWT <- function(sequence){
  sequence  <- str_c(sequence, '$')
  sequences <- c(sequence)
  n         <- nchar(sequence)

  for (i in 1:(n-1)){
    sequence <- str_c(str_sub(sequence, 2, n),
                     str_sub(sequence, 1, 1))
    sequences <- c(sequences, sequence)
  }
  sequences <- sort(sequences) 
  
  bw.matrix           <- data.frame(matrix(, n, n))
  colnames(bw.matrix) <- 1:n

  for (i in 1:n){
    bw.matrix[i, ] <- strsplit(sequences[i], split = '')[[1]]
  }
  return(paste(bw.matrix[,n], collapse = ''))
}
```


```{.r .numberLines}
BWT('GATTACA')
#> [1] "ACTGA$TA"
```

One can verify that this output is equal to result we obtained earlier.

Out of pure curiosity lets check the Burrows-Wheeler transform for a longer sequence.


```{.r .numberLines}
BWT('ATGCTCGTGCCATCATATAGCGCGCGCGCGATCTCTACGCGCG')
#> [1] "GTTTCCG$TCGGGGGAGGGTTGTCCTCCCCCCATCCAAACCAGA"
```

Please note that the input string has no identical characters at adjacent positions, whereas in the transformed sequence such situation appears quite often. These substrings of identical characters will allow us represent the sequence in a more condensed manner and expediate pattern matching.
