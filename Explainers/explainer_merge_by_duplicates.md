### One-minute summary

In general, be cautious when merging tables if there are duplicates in a
shared `by`-column! The behavior of `merge()` is not obvious when this
is the case.

If you use `merge()` to bind two tables together by a shared ID column,
and end up with a table with more rows than there are unique IDs when
using the `all = TRUE`, `all.x = TRUE`, or `all.y = TRUE` arguments, it
is likely because there are duplicate rows for one or more IDs in one of
the tables (depending on which of `all` / `all.x`/ `all.y` you are
using). Pivot the table with duplicate values to wide format (so each ID
can have more than one “copy” of the original columns), remove the
duplicates, or aggregate the table somehow so only one row remains per
ID. You could also keep the duplicates, and if so you *could* add
another ID column to distinguish them from each other, but you don’t
necessarily *have* to.

``` r
length(unique(table$ID)) < nrow(table) # TRUE if n. of unique IDs < n. of rows
```

You can figure out if a table has duplicate values in the ID column like
this.

### Background

A student at R Club noticed that the `merge()` function works in an
unintuitive way. The student’s goal was to add a number of columns from
one table to another table - the tables had a shared ID column to use as
reference when placing values in the added columns. But when merging the
two tables with `merge()`, the combined table somehow had more rows than
there were unique IDs. This is not the behavior the student expected.
Why would code that was just supposed to add columns from one table to
another by a shared ID column end up producing a table with more rows
than there were IDs in the first place?

At R Club, I struggled to see why that would happen too. In this
document, we will understand together how it happens. We will also take
a look at what you can do to avoid or work around it.

### Recreate the problem

Let’s think of a silly but easy-to-follow example with small tables that
we can see all of easily.

#### Making some tables

We first create two tables, `letters_uppercase` and `letters_lowercase`,
that we can merge together later. They should have the first five
letters of the alphabet in uppercase (A - E) and lowercase (a - e),
respectively. `letters_uppercase` will have a column called
`letter_uppercase`, and `letters_lowercase` will have a column called
`letter_lowercase`. Both tables should have a factor (i.e. categorical)
column `number` with the position of each letter in the alphabet, so 1
to 5, as A - E are the first five letters in the alphabet.

Later, we want to merge these into a table containing the letters of the
alphabet in both upper and lower case. As a twist, to convince ourselves
that we can merge the `letter_uppercase` and `letter_lowercase` values
according to their shared `number`, and not just by their order in the
table (which you could do with the function `cbind()`, by the way), we
order `letters_lowercase` randomly.

``` r
numbers <- seq(1, 5)                       # Numbers 1 - 5 in order
letters <- c("A", "B", "C", "D", "E")      # Letters A - E in order

numbers_random <- sample.int(5, 5)         # Numbers 1 - 5 in random order
letters_random <- letters[numbers_random]  # Letters A - E in order of numbers_random

letters_uppercase <- data.frame(
  number    = as.factor(numbers),          # The numbers are labels, not "real numbers"
  uppercase = letters
)

letters_lowercase <- data.frame(
  number    = as.factor(numbers_random),
  lowercase = tolower(letters_random)      # make letters lowercase with tolower()
)
```

    ##   number uppercase
    ## 1      1         A
    ## 2      2         B
    ## 3      3         C
    ## 4      4         D
    ## 5      5         E

    ##   number lowercase
    ## 1      3         c
    ## 2      5         e
    ## 3      4         d
    ## 4      2         b
    ## 5      1         a

#### Combining the tables

We can now combine the upper and lower case letters using `merge()`. We
want the values from the `letter_uppercase` and `letter_lowercase`
columns to be merged together according to the `number` column, so we
use the `by` argument in `merge()`.

``` r
letters <- merge(
  x = letters_uppercase,
  y = letters_lowercase,
  by = "number"
)
```

    ##   number uppercase lowercase
    ## 1      1         A         a
    ## 2      2         B         b
    ## 3      3         C         c
    ## 4      4         D         d
    ## 5      5         E         e

#### Finding the problem

So far this works as expected. Let’s make a new table `words`, with some
example words beginning with the letters. We’ll make one word in upper
case and one in lower case. We’ll only do it for A/a and C/c, the
letters with number 1 and 3.

``` r
words <- data.frame(
  number = c(1, 1, 3, 3),
  word = c(
    "Aperol",
    "apple",
    "Campari",
    "coconut"
  )
)
```

    ##   number    word
    ## 1      1  Aperol
    ## 2      1   apple
    ## 3      3 Campari
    ## 4      3 coconut

Okay, now we want to bind these words to the table with letters!

``` r
letters_and_words <- merge(
  x = letters,
  y = words,
  by = "number"
)
```

    ##   number uppercase lowercase    word
    ## 1      1         A         a  Aperol
    ## 2      1         A         a   apple
    ## 3      3         C         c Campari
    ## 4      3         C         c coconut

Well, that’s not what we wanted. Now we only keep the letters 1/A/a and
3/C/c. And there are two rows for each now? **It actually isn’t obvious
how we should add two rows for each of 1 and 3 to just one row even if
we say `by = "number"`! In fact, with this minimal example, it’s quite
obivous why there has to be two rows for number 1 and 3 now when we use
`by = "number"`!** But why would it still drop the other numbers? We can
use `all = TRUE` to keep all the rows from `letters` …

``` r
letters_and_words <- merge(
  x = letters,
  y = words,
  by = "number",
  all = TRUE
)
```

    ##   number uppercase lowercase    word
    ## 1      1         A         a  Aperol
    ## 2      1         A         a   apple
    ## 3      2         B         b    <NA>
    ## 4      3         C         c Campari
    ## 5      3         C         c coconut
    ## 6      4         D         d    <NA>
    ## 7      5         E         e    <NA>

Okay! This makes sense - but we can see what problem the student in R
Club must have had. **If there are duplicate values you don’t know of in
the `by`-column, the table length (number of rows) will grow by that
number of duplicates.** I didn’t realize this in R Club, because we
didn’t think there were duplicates IDs in the data. And it’s not that
weird that the student didn’t know, because it’s obvious here with a
minimal example, but not obvious when there are thousands of rows in a
table and it’s not clear why there should be duplicates. Well. Now we
know - can you check your own tables for duplicates, and what can you do
about it if there are?

### Fixing the problem

#### Check for duplicates

This code checks whether we have duplicates in the column we use as IDs
to feed the `by`-argument in `merge()`. We of course know there are
duplicates, but this is just an example of how you could do it for your
own tables.

``` r
length(unique(words$number)) < nrow(words) # TRUE if n. of unique IDs < n. of rows
```

    ## [1] TRUE

This comes out TRUE, because there are more rows than there are unique
numbers, so we know there are duplicates.

#### Pivot wide

One way you can deal with duplicate rows is that you can turn the extra
data into columns. You might actually need to add more information to
the table first to do this! Here’s how it works.

We could add some information about the case of the words to the `words`
table and then pivot it wider using the function `pivot_wider()` from
the `tidyr`-package. If you have multiple observations of one individual
in a study, you could for instance add a column `observation_number` to
separate observations and then turn the “extra” rows for each ID into
columns.

This is roughly what to do, and how it solves the problem in our
example:

``` r
words <- cbind(
  words,
  case = c("upper", "lower", "upper", "lower") # Add information about case of word
)
```

    ##   number    word  case
    ## 1      1  Aperol upper
    ## 2      1   apple lower
    ## 3      3 Campari upper
    ## 4      3 coconut lower

``` r
words <- tidyr::pivot_wider(
  words,
  names_prefix = "word_", # Prefix for our new column names
  names_from = "case",
  values_from = "word"
)
```

    ## # A tibble: 2 × 3
    ##   number word_upper word_lower
    ##    <dbl> <chr>      <chr>     
    ## 1      1 Aperol     apple     
    ## 2      3 Campari    coconut

``` r
letters_and_words <- merge(
  x = letters,
  y = words,
  by = "number",
  all = TRUE
)
```

    ##   number uppercase lowercase word_upper word_lower
    ## 1      1         A         a     Aperol      apple
    ## 2      2         B         b       <NA>       <NA>
    ## 3      3         C         c    Campari    coconut
    ## 4      4         D         d       <NA>       <NA>
    ## 5      5         E         e       <NA>       <NA>

#### Remove the duplicates

We can also remove duplicates with the function `distinct` from the
package `dplyr`. You might not want to remove your duplicates though, if
they are real data that you want to keep working with! You should
consider whether you can find some way to aggregate the data in the
duplicates, or pivot the dataset wider like in the section above.

``` r
# This is the original definition of the words table again
words <- data.frame(
  number = c(1, 1, 3, 3),
  word = c(
    "Aperol",
    "apple",
    "Campari",
    "coconut"
  )
)

words <- dplyr::distinct(words, number, .keep_all = TRUE) #.keep_all columns
```

    ##   number    word
    ## 1      1  Aperol
    ## 2      3 Campari

#### Maybe it’s okay to have duplicates?

It may also be that you want to keep duplicates! If you work in animal
ecology, for example, perhaps you have field observations of the same
animal on multiple days. In that case, you could still bind columns of
genotype information from a different table to multiple observations of
the same animal from different days! In the end, this is something to
consider in context of the research questions you are asking and the
analyses you want to implement. Duplicates could be a good thing, and
now you know how to figure out, if `merge()` behaves in an unexpected
way!
