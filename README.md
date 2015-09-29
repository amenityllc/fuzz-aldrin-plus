## What is fuzzandrin-plus ?

- A fuzzy search / highlight that specialize for programmer text editor. It try to provide intuitive result be recognizing patterns in string that people use to reason with.

- A rewrite of the fuzzaldrin library. API is backward compatible with some extra options. Tuning has been done from report usage of the Atom text editor.

- At this point of time, it may be merged back into fuzzaldrin or live as it's own package, we'll see.


## What Problem are we trying to solve ?

### Score how matched characters relates to one another.

- One of the most complained thing is not being able to find exact match.
- A great source of questionable results come from scattered character, spread a bit randomly in the string.

And we plan to address those issues by scoring run of consecutive characters.
Exact match will be a special case where the run is 100% of the query length.

Up to now match length was used as a proxy for quality. This work reasonably well when subject is a single word, but break when subject contain multiple words, for example see:

- **Core**
- **Co**nt**r**oll**e**r
- Extention**Core**

In `Core` vs `Controller` size is a good indicator of quality, but not so much in `Controller` vs `ExtentionCore`. This is because match compactness mater more than haystack size. Match compactness is the principle behind scoring of the Selecta project.


#### Run length / consecutive

So far the examples can be handled by an `indexOf` call. However there are times where a single query can target multiple part of an candidate.

For example when candidate contain multiple words
- `git push` vs `Git Plus: Push`
- `email handler` vs `email/handler.py`

Another example is to jump above common strings.
- `toLowerCase`
- `toLocaleString`
- `toLocalLowerCase`

We could use a query like `tololo`  to select the third option of these.


### Select character based on score.

The current algorithm always select the first available matching character (leftmost alignment) then try to identify how it should be scored. The problem is that the most interesting instance of a character is not necessarily on the left.

For example on query `itc`, we should match
-  **I**mportance**T**able**C**trl.

Instead leftmost aligement miss the acronym pattern:
- **I**mpor**t**an**c**eTableCtrl.

For query `core` against `controller_core` leftmost alignment miss the consecutive run:
- **co**nt**r**oll**e**r_core

To handle this we propose to setup the max run-length scoring inside an optimal alignment scheme. (Implemented for example using dynamic programming)

### Prevent Accidental acronym.

Fuzzladrin handle acronym by giving a large per character bonus.  Currently a start of word bonus match almost as much as 3 proper case character (or 7 wrong case ones!)

For query install "install" should result be in this order ?
- F**in**d & Replace **S**elec**t** **All**
- Application: **Install**

Here we have '**S**elect **A**ll' boost the score of the first candidate because we match two word-start VS only one for 'install'.

For query "git push", should we order result in that order ?
- "**Git** **P**l**u**s: **S**tage **H**unk"
- "**Git** Plus: **Push**"

What about the fact we match 3 start-of-word in `Plus Stage Hunk` ? PSH is very close to '**p**u**sh**'.

That kind of question arise even more often with optimal selection because the algorithm will lock on those extra acronym points.

What we propose in this PR is that word start-of-word character only have a moderate advantage by themselves. Instead they form strong score by making an acronym pattern with other start-of-word character.

For example with query `push`:
- match against `Plus: Stage Hunk`, we have P + u + SH so group of 1+1+2
- match against `push` single group of 4.
- The substring win for having the largest group

For example with query `psh`:
- match against `Plus: Stage Hunk`, we have PSH so group of 3
- match against `push`, we have p+sh so group of 1+2
- The acronym win for having the largest group.


This way we can score both substring and acronym match using the structure of the match. We'll refine the definition of consecutive acronym later.


### Score the context of the match.

Some people proposed to give perfect score to exact case sensitive match. This can be understood because exact match and consecutive are two area where fuzzaldrin is not great.

However should 'sw**itc**h.css' be an exact match for `itc` ?
Even when we have **I**mportance**T**able**C**trl available ?

Few people will argue against "diag" preferring "diagnostic" to "Diagnostics".

However should `install` prefer "Un**install**" over "**Install**" ? Or should it be the other way around ?  This is a case of case-sensitive match vs start-of-word ...


### Manage the multiples balances of path scoring.

We want to prefer match toward the start of the string.
Except we also want to prefer match in the file-name (which happens near the end of the string)

We want to prefer shorter and shallower path.
Except we want to retrieve some deeper file when file name is clearly better.

We want to prefer match in the file-name
Except when the query describe very well the full path.
(see query `modeluser` vs `models/user.rb` or `moderator_column_users.rb`)


## Proposed scoring


### 1. Characters are chosen by their ability to form pattern with others.

Pattern can be made of
 - consecutive letter of the subject,
 - consecutive letter of the Acronym of subject.

Start-of-word (acronym) character are special in that they can either make pattern with the rest of the word or other acronym character.

- This replace a situation where acronym character had unconditional large bonus. Now the bonus is still large but conditional to being part of some pattern.

- CamelCase and snake_case acronym are treated exactly the same. They will however get different point for matching uppercase/lowercase query


### 2. Primary score attribute is pattern length.

- A pattern that span 100% of the query length is called an exact match.
 	- There is such a thing as an acronym exact match.

- Because every candidate is expected to match all of query larger group should win against multiple smaller one.
    - Rank of a candidate is the length of it's largest pattern.
    - When every pattern in a candidate are larger than pattern in a second one, highest rank candidate is said to be dominant.
        - 1 group of 6 >  2 group of 3 > 3 group of 2.

	- When some group are larger and some are smaller, highest rank match is said to be semi-dominant.
         - group of 4+2 vs 2 groups of 3. Group of 4 wins agains the first group of 3, group of 2 loose against second group of 3.

### 3. Secondary score attribute is match quality.

- Match quality is made of proper-casing and context score
    - Main role of match quality is to order candidate of the same rank.
    - When match is semi-dominant match quality can overcome a rank-1 difference.

- Context score consider where does the match occurs in the subject.
	- Full-word > Start-of-word > End-of-word > Middle-of-word
	- On that list, Acronym pattern score at Start-of-word level. (That is just bellow full-word)

- Proper case is both gradual and absolute.
   - The less error, the better
   - 100% Case Error will be called wrong-case, for example matching a CamelCase acronym using lowercase query.  
   - Exactly 0 error is called CaseSentitive or ExactCase match.
     They have a special bonus smaller than start-of-word bonus but greater than end-of-word bonus.
     - This allow a start-of-word case sentitive match to overcome a full-word wrong-case match.
     - Also allow to select between a lowercase consecutive and CamelCase acronym using case of query.
     - Make "Installed" win over "Uninstall" because  start-of-word > Exact Case.

- **Q:** Why can't you simply add extra length. For example add an extra virtual character for a start-of-word match.
   - **A:** We cannot do that on partial match because then the optimal alignment algorithm will be happy to split word and collect start-of-word bonus like stamp. (See accidental acronym)

- **Q:** Why do you still do it on exact match ?
   - **A:** First once you have matched everything there's no danger of splitting the query, then it's there for exact match to bubble up, despite longer/deeper path. If after more test/tuning we realize it's not needed, we'll be happy to remove it, the less corner case the merrier.

- **Q:** Why are you using lowecase to detect CamelCase ?
  - **A** CamelCase are detected as a switch from lowercase to UPPERCASE. Defining UPPERCASE as not-lowercase, allow case-invariant character to count as lowercase.   A lot of case invariant character are also separator, some are not such as number, and symbol such as `:!=<>()`

### 4. Tertiary score attributes are subject size, match position and directory depth

- Mostly there to help order match of the same rank and match quality, unless the difference in tertiary attribute is large.
 	-(Proper definition of large is to be determined using real life example)

- In term of importance of effect it should rank start-of-string > string size > directory depth.

## Score Range

- **Score is 0 if and only if there is no match.**

- Otherwise it is strictly positive integer.

- The maximum range is `score(query,query)` whatever that number is. Longer query will have greater maximal score.

- Score exist mainly to be compared with other score of the same query and implement scoring rule described above. 

- Score have a high dynamic range and consider a lot of information. Equality is unlikely. For that reason, **multiplicative bonus should be preferred over additive ones**.

-------------

### Acronym Prefix
More detail on acronym match


The acronym prefix is a group of character that are consecutive in the query and sequential in the acronym of the subject.
That group start at the first character of the query and end at the first error (character not in acronym).
If there's no error we have an acronym exact match (100% of query is part of acronym)


**For example if we match `ssrb` against `Set Syntax Ruby` we'll score it like so**

````
  012345678901234
 "Set Syntax Ruby"
 "000 SSR0b0 0000"
````

- Acronym scored as start-of-word consecutive rank 3 + an isolated letter.
- Here we have a wrong-case match. "SSRb" or "SSRB" would have case-sensitive points on the acronym pattern (case of isolated letter is not important)
- Position of the equivalent consecutive match is the average position of acronym characters.
- Size of the candidate does not chance.


**Another example is matching `gaa` against `Git Plus: Add All` we'll score it like so**

````
  01234567890123456
 "Git Plus: Add All"
 "000000 GAA00 0000"
````

- here we conveniently allow to skip the `P` of `Plus`.


**Then what about something like "git aa" ?**

This is a current limitation. We do not support acronym pattern outside of the prefix. Mostly for performance reason.
Acronym outside of the acronym prefix will have some bonus, scoring between isolated character and 2 consecutive.
There are multiple thing we can improve if one day we implement a proper multiple word query support, and this is one of them.

### Optional characters

Legacy fuzzaldrin had some support for optional character (Mostly space, see `SpaceRegEx`). Because the scoring do not support errors, the optional character where simply removed from the query.

With this PR, optimal alignment algorithm support an unlimited number of errors. The strict matching requirement is handled by a separate method `isMatch`. The optional character implementation is done by building a subset of query containing only non-optional character (`coreQuery`) and passing that to `isMatch`.

This new way of doing thing means that while some characters are optional, candidate that match those characters will have better score. What this allow is to add characters to the optional list without compromising ranking.

Character that has been added include `-` and `_` because multiple  specs require that we should threat them as space. Also `\` and `:` to support searching a file by PHP and Ruby name-space. Finally `/` to mirror `\` and support a better work flow in a multi-OS environment.

Finally option `allowErrors` would make any character optional. Expected effect of that would be spell-checker like, but slower match.


### Path scoring

- Score for a path is made of about half score of full path and half score of basename.

- By default basename is what follow the last slash (after trailing slashes has been removed)

- Exact balance between full path and basename is set by the directory depth: for deeper directory there is less retrieval effect (importance of basename) 

- Full path is penalized twice for size. Once for it's own size, then a second time for size of the basename. This address various demand for shorter basename to win. Extra basename penalty is dampened a bit.

- Basename is scored as if `allowErrors` was set to true. (Full-path must still pass `isMatch` test) This allow to support query such as `model user` against path `model/user`. Previously, the basename score would be 0 because it would not find `model` inside basename `user`. Variable `queryHasSlashes` sort of addressed this issue, but was inconsistent with usage of `<space>` as folder separator

- When query has slashes (`path.sep`) the last or last few folder from the path are promoted to the basename. (as many folder from the path as folder in the query)

-------------

## Algorithm (Optimal alignment)

### LCS: Dynamic programming Table
Let's compare  A:`surgery` and B:`gsurvey`.
To do so we can try to match every letter of A against every letter of B.

This build a 2D Table.
- The match start at [0,0] trying to compare first letter of each.
- The match end at [m,n] comparing last letter of each.

At each position [i,j] the best move can be one of the 3 options.

- match `A[i]` with `B[j]` (move diagonal, add 1 to score)
- skip `A[i]` (move left, copy score)
- skip `B[j]` (move down, copy score)

We don't know which one of these 3 is the best move until we reach the end, so we record the score of best move so far. The last cell contain the score of the best alignment. If we want to output that alignment we need to rebuild it backward from the last cell.

````
    s u r g e r y
 g [0,0,0,1,1,1,1]  : best move is to align `g` of *g*survey with `g` of  sur*g*ery, score 1
 s [1,1,1,1,1,1,1]  : we can align `s`, but doing so invalidate `g`. Both score 1, we cannot decide
 u [1,2,2,2,2,2,2]  : if we align s, we can also align u, we have a winner
 r [1,2,3,3,3,3,3]  : we can align `r`
 v [1,2,3,3,3,3,3]  : nothing we can do with that `v`, score stay the same
 e [1,2,3,3,4,4,4]  : we can align `e`  (we skipped `g` the same way we skipped `v`)
 y [1,2,3,3,4,4,5]  : align y (we skipped `r` )
````

**Best alignment is**

````
gsur-ve-y
-|||--|-|
-surg-ery
````

For those familiar with code diff, this is basically the same problem. Except we do alignment of character in a word and diff perform alignment of lines in a file. Character present in second word but not the first are addition, character present only in the first are deletion, and character present in both are match - like unchanged lines in a diff.

To get that alignment we start from the last character and trace back the best option. 
The pattern to looks for an **alignment** is the corner increase (diagonal+1 is greater than left or up.)

````
4,4   3,3   2,2    1,1    0,0
4,5   3,4   2,3    1,2    0,1
````

- (There is an implicit row and column of 0 before the matrix)

The pattern to look for to **move left** is:

````
3,3
4,4
````

The pattern to look for to **move up** is:

````
3,4
3,4
````

We try to resolve equality the following way:

````
3,3
3,3
````

1. Prefer moving UP: toward the start of the candidate. This will highlight start of string instead of the end.
2. If not available, prefer moving LEFT (optional character)
3. Only accept alignment DIAG when it's the absolute best option.




### Algorithm Conclusion

LCS allow to detect witch character of query are common to both word while being in proper order (for example g is common to both word but discarded because out of order.)

LCS is not immediately useful. This is because fuzzaldrin require ALL of query to be in subject, else the score is 0. So all non zero score would be the same.

But the tool used to solve LCS is very useful to our need.
The ability to select the best path and skip that `g` even if it is present is the key to improve over left-most alignment. (Skip one instance, select another one - or, in the case of optional character, none)

### Similarity score

LCS {0,1} score scheme ask the question is there a match for this character ? yes/no. However there could be different quality of match. For example variation of characters `e`, `E`, `é`, `è`, handwriting or phonetic similarity.

We use character similarity as a way to form and score patterns. That is, we consider that character are similar from their own quality (case) as well of being part of similar neighbourhood (consecutive letters or acronyms)

This is a bit of a cheat after so much talk about how we should focus on pattern rather than characters, but it allow the granularity to form one pattern instead of another.

There are some rules that limit our scoring ability (for example we cannot go back in time and correct the score based on future choice) but overall that scheme is very flexible.

### Where is the matrix ?

While the above matrix describe the computation, we do not need to store the whole matrix when we only request the score. This is because while computing any cell we only need 3 other previously computed cell: UP, LEFT and DIAG.

Suppose we process the cell [3,5]

20, 21, 22, 23, 24, 25, 26, *27, 28, 29*   
*30, 31, 32, 33, 34,* **35**, 36, 37, 38, 39  

To build that score we only need values 24(DIAG), 25(UP), 34(LEFT).
So instead of a whole matrix we can keep only the two current lines.

Furthermore, anything on the left of 24 on the first line is not needed anymore. Also on the second line, anything to right of 35 has not been computed yet. So we can build a more compact structure using one composite row + one diagonal.

score_diag =  24    
score_row = 30, 31, 32, 33, 34, 25, 26, 27, 28, 29

#### Preparing next value

Once we have computed the value of the cell [3,5], we can insert it in the structure, taking care of saving next diagonal before overwriting it.

diag =  25    
row = 30, 31, 32, 33, 34, **35**, 26, 27, 28, 29

To compute value of cell [3,6] we take 
- UP value (26) from the row.
- DIAG value, from the diag register. 
- LEFT value from the previously computed value: 35

### Initial values

Before entering the matching process, the row is initialized with 0. Before scoring each row, the LEFT and DIAG register are reset to 0.

This has the effect of placing a virtual row and column of 0 before the matrix. And allow to deal with boundary condition without any special case.

### Memory management

We setup row vector to contain the query. So scoring a query of size 5 against a path of size 100, would require a `5x100 = 500` cells matrix. Instead we use a 5 item row + some registers. This should ease memory management pressure.

Each character of the query manage it's own best score. Or more precisely. Each cell `row[j]` manage the best score so far of matching `query[0..j]` until query is fully matched.

### Consecutive Score (Neighbourhood) Matrix.

We cache the consecutive score in a virtual matrix of the same size of score.
We also use the composite row scheme to do that.

In highlight, (match) we make the consecutive bonus conditional to not breaking the chain before:

For example query `abcdz` vs subject `abcdzbcdz`. Between `abcd` and `bcdz`, `abcd` will win for being sooner in string. Now between the two `z`, the first one is isolated and the second one is part of a rank 4 group. However given that `bcd` are matched sooner the second z is really an isolated match, so the first one would win.

For performance reason this is not done for filtering. However it produce not so intuitive result in highlight and thus it is addressed.

-------------

## Performance

Let's consider the following autocomplete scenario.
- Symbol bank have 1000 items.
- User receive about 5 suggestion for it's query.
- Of those 5, 1 is a case sensitive exact match.
- That particular user almost always want that case sensitive match.

Should we optimize for case sensitive `indexOf` before trying other things ?
Our answer to that question will be no. Case sensitive exact match are valuable because they are rare.
Even if the user try to get them, for each one of those we have to reject 995 entry and deal with 4 other kind of match.

And that is our first principle for optimization: **Most of the haystack is not the needle**.
Because rejection of candidate happens often we should be very fast at doing that. 

Failing a test for case sensitive `indexOf` tell us exactly nothing for case-insensitive `indexOf` or acronym, or even scattered letters.
That test is too specific. To reject match efficiently we should aim for the lowest common denominator: scattered case-insensitive match.

And that is exactly the purpose of `isMatch`

### Most of the haystack is not the needle

That sentence is true at the candidate level, but also at the character level. 

**Let's consider this line: `if (subject_lw[i] == query_lw[j])`**
This test for match points (or hits). It refer to the `diag+1` in the above algorithm description, with the `+1` being refined to handle different level of character and neighbourhood similarity.


**How often is that condition true ?**
Let's consider an alphabet that contain 26 lowercase letters, 10 numbers, a few symbols ` _!?=<>`. That's a 40+ symbol alphabet. Under a uniform usage model of those symbols, we have the hit condition occurs about 2.5% of the time (1/40).
If we suppose only 10-20 of those character are popular the hit rate is about 5-10%.

This mean we'll try to minimize the number of operation that happens outside of math points. Increasing the cost of a hit, while deceasing the cost of non hits looks like a possibly worthwhile proposition. 

A connonical example of this is that, instead of testing each character against the list of separator, setting a flag for next character being a start-of-word, we first comfirm a match then look behind for separator. The characterisation works is sometime repeated more than once, but so far this scheme benchmarked better than alternatives we have tried to avoid doing extra work.

Having work concentrated at hit points is also a natural fit to our logic, the most expensive part being to determine how to score similarity between characters (including context similarity). However, it also means we'll want to somewhat control the number of positive hit we'll compute - that's the purpose of missed hit optimisation.


### What about stack of needles ?

To the extend the user is searching for a specific resource this should be uncommon.

It can still happens in some situation such as:
 - Search is carried as user type (the query is not intentional)
 - Intentional query is not fully typed, match-all is a temporary step.

One way to deal with that is to not use the full matching algorithm when we can deal with something simpler.
Example `indexOf` instance. In an ideal world we would not use it because corner cases cause problem. But the next best option is to use exactly the same scoring algorithm for consecutive in optimal alignment and `indexOf`.

One special note: Acronym still have to be checked even if we have an exact match.

For example query `su` against `StatusUrl`. As an exact match it's poor: 'Statu**sU**rl' middle of word and wrong case.
However as an acronym it's great: '**S**tatus**U**rl'. That motivated us to create the specialized `scoreAcronyms`

What is nice is that while `scoreAcronyms` was created to speed up exact search, it also provided very valuable information for accuracy.
It latter became a corner stone in the processing of accidental acronym. 

The result is that for exact match and exact acronym match we bypass the optimal alignment algorithm, giving very fast results.
We still have to deal with fuzzier stack of needles and the next two optimization address this.

### Hit Miss Optimization.

A hit occurs when character of query is also in the subject.
 - Every (i,j) such that subject[i] == query[j], in lowercase.

A missed hit occurs when a hit does not improve score.

To guarantee optimal alignment, every hit has to be considered.
However when candidate are long (deep path) & query contains popular character (vowels) , we can spend a huge amount of time scoring accidental hit.

So we use number of missed hit as a heuristic for unlikely to improve.
Let's score `itc` vs `ImportanceTableControl`

- `I` of `Importance`: First occurrence, improve over none.
- `t` of `Importance`: First occurrence, improve over none.
- `c` of `Importance`: First occurrence, improve over none.
- `T` of `Table` : Acronym match, improve over isolated middle of word.
- `C` of `Control` : Acronym match, improve over isolated middle of word.
- `t` of `Control`: no improvement over acronym `T`: first hit miss.

- After a certain threshold of hit miss we can consider it's unlikely the score will get better
- Despite above example hit miss optimization do not affect scoring of exact match (sub-string or acronym)
- There are some legitimate use for hit miss, for example while scoring query `Mississippi` each positive match for `s` or `i` may trigger up to 3 hit miss on the other occurrence of that letter in query.

- For that reason we propose counting consecutive hit miss and having maximum one hit miss per character of subject.

**Q:** Does this grantee improvement over leftmost alignment ?
**A:** It'll often be the case but no guarantee on pathological match.
 For example, in query `abcde` against candidate '**abc**abcabcabcabcabcabc_de' we may trigger the miss count before matching `de`. It'll still be registered as a match and probably a good one with `abc` at the start, `de` will be scored as optional characters not present.

Candidate  'abcabcabcabcabcabc**abcde**' will not have any problem because it does not affect exact match.

Real example is searching `index` in the benchmark. Where `i`, `n`, `d`, `e` exist scattered in folder name, but x exist in the extension `.txt`. However the whole point of this PR is to prefer structured match to scattered one so this might not be a problem.

### High Positive count mitigation
**[option `maxInners`, disabled by default]**

A lot of the speed of this PR come from the idea that rejection happens often and we need to be very efficient on them to offset slower higher quality match. Unfortunately some query will match against almost everything.

- Fast short-circuit path for exact substring acronym help a lot.
- Missed hit heuristic also help a lot for general purpose match.

However we may still be too slow for interactive time query on large data set. This is why `maxInners` option is provided.

This is the maximum number of positive candidate we collect before sorting and returning the list.

The realization is that a query that match everything on a 50K item data set is unlikely to show anything useful to the user above the fold (say in the first 15 results).

So then the priority is to detect such case of low quality (low discrimination power) query and report fast to the user so user can refine its query.

A `maxInners` size of about 20% of the list works well. It is not needed on smaller list.

### Active Region Optimization

Before the first occurrence of the first char of query in the subject, or after the last occurrence of the last char of query in the subject it is impossible to make a match. So we'll trim the subject to that active region. The search for those boundary is linear while the optimal alignment algorithm is quadratic so it is an improvement however little or large we move.

### Benchmark
- All test compare this PR to previous version (legacy)

- The first test `index` is a typical use case, 10% positive, 1/3 of positive are exact match.
  - We are about 2x faster

- Second test `indx` remove exact matches. Just under 2x faster

- Third test `walkdr`, 1% positive, mostly testing `isMatch()`, above 2x faster.

- Fourth test `node`, exact match, 98% positive, bit under 2x faster.

- Test 5 `nm`, exact acronym match, 98% positive, about 10% slower.

- Test 6 `nodemodules` is special in that it use a string that score on almost every candidate, often multiple time per candidate and individuals characters are popular. It also avoid exact match speed-up. About 2x slower, but unlikely to happens in real life. `maxInners` mitigation cover that case.


````
Filtering 66672 entries for 'index' took 62ms for 6168 results (~10% of results are positive, mix exact & fuzzy)
Filtering 66672 entries for 'index' took 120ms for 6168 results (~10% of results are positive, Legacy method)
======
Filtering 66672 entries for 'indx' took 69ms for 6192 results (~10% of results are positive, Fuzzy match)
Filtering 66672 entries for 'indx' took 126ms for 6192 results (~10% of results are positive, Fuzzy match, Legacy)
======
Filtering 66672 entries for 'walkdr' took 30ms for 504 results (~1% of results are positive, fuzzy)
Filtering 66672 entries for 'walkdr' took 70ms for 504 results (~1% of results are positive, Legacy method)
======
Filtering 66672 entries for 'node' took 112ms for 65136 results (~98% of results are positive, mostly Exact match)
Filtering 66672 entries for 'node' took 213ms for 65136 results (~98% of results are positive, mostly Exact match, Legacy method)
======
Filtering 66672 entries for 'nm' took 60ms for 65208 results (~98% of results are positive, Acronym match)
Filtering 66672 entries for 'nm' took 56ms for 65208 results (~98% of results are positive, Acronym match, Legacy method)
======
Filtering 66672 entries for 'nodemodules' took 602ms for 65124 results (~98% positive + Fuzzy match, [Worst case scenario])
Filtering 66672 entries for 'nodemodules' took 123ms for 13334 results (~98% positive + Fuzzy match, [Mitigation])
Filtering 66672 entries for 'nodemodules' took 295ms for 65124 results (Legacy)
````

**Q:** My results are not as good.
**A:** Run the benchmark a few time, it looks like some optimization kick in later. (Or CPU on energy efficient device might need to warm up before some optimization are activated)



## Prior Art

[Chrome FilePathScore](https://chromium.googlesource.com/chromium/blink/+/master/Source/devtools/front_end/sources/FilePathScoreFunction.js#70)

[Textmate ranker](https://github.com/textmate/textmate/blob/master/Frameworks/text/src/ranker.cc#L46)

[VIM Command-T ](https://github.com/wincent/command-t/blob/master/ruby/command-t/match.c#L22)

[Selecta](https://github.com/garybernhardt/selecta/blob/master/selecta#L415)

[PeepOpen](https://github.com/topfunky/PeepOpen/blob/master/Classes/Models/FuzzyRecord.rb)

[flx](https://github.com/lewang/flx)


## List of addressed issues

### Exact match vs directory depth.

https://github.com/atom/fuzzaldrin/issues/18
-> actionsServiceSpec

https://github.com/atom/atom/issues/7783
-> usa_spec

### Start of string VS directory depth

https://github.com/atom/fuzzy-finder/issues/57#issuecomment-133531653
-> notification

https://github.com/atom/fuzzy-finder/issues/21#issuecomment-48795958
-> video backbone

### Folder / file query

https://github.com/atom/fuzzy-finder/issues/21#issue-29106280
-> src/app vs destroy_discard

https://github.com/atom/fuzzy-finder/issues/21#issuecomment-46920333
-> email handler

https://github.com/substantial/atomfiles/issues/43
-> model user

### Spread/group vs directory depth

https://github.com/atom/fuzzy-finder/issues/21#issuecomment-138664303
-> controller core

### Initialism

https://github.com/atom/fuzzy-finder/issues/57#issue-42120886
-> itc switch / ImportanceTableCtrl

https://github.com/atom/fuzzy-finder/issues/57#issuecomment-95623924
-> application controller

https://github.com/atom/fuzzaldrin/issues/21
-> fft vs FilterFactorTests

### Accidental Acronym

https://github.com/atom/command-palette/issues/28
-> Install / Find Select All

https://github.com/atom/fuzzaldrin/issues/20#issue-93279352
-> Git Plus Stage Hunk / Git Plus Push


### Case sensitivity

https://github.com/atom/autocomplete-plus/issues/42
-> downloadThread / DownloadTask

https://github.com/atom/fuzzaldrin/issues/17
-> diagnostics / Diagnostic


### Optional Characters

https://github.com/atom/fuzzy-finder/issues/91
https://github.com/atom/fuzzaldrin/issues/24

-> PHP Namespaces, let "\" match "/"
-> (would be nice for config file in mized OS environment too)

https://github.com/atom/fuzzy-finder/pull/51

-> Ruby Namespaces, let "::" match "/"

https://github.com/atom/fuzzy-finder/issues/10
-> SpaceRegex, let " " match "/"
-> was already implemented, posted here to show parallel.


### Suggestions

https://github.com/atom/fuzzy-finder/issues/21#issue-29106280
-> we implement suggestion of score based on run length
-> todo allow fuzzaldrin to support external knowledge.
