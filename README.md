# Pagerank Project

In this project, you will create a simple search engine for the website <https://www.lawfareblog.com>.
This website provides legal analysis on US national security issues.

**Due date:** Sunday, 18 September at midnight

**Computation:**
This project has low computational requirements.
You are not required to complete it on the lambda server (although you are welcome to if you'd like).

## Background

**Data:**

The `data` folder contains two files that store example "web graphs".
The file `small.csv.gz` contains the example graph from the *Deeper Inside Pagerank* paper.
This is a small graph, so we can manually inspect the contents of this file with the following command:
```
$ zcat data/small.csv.gz
source,target
1,2
1,3
3,1
3,2
3,5
4,5
4,6
5,6
5,4
6,4
```

> **Recall:**
> The `cat` terminal command outputs the contents of a file to stdout, and the `zcat` command first decompressed a gzipped file and then outputs the decompressed contents.

As you can see, the graph is stored as a CSV file.
The first line is a header,
and each subsequent line stores a single edge in the graph.
The first column contains the source node of the edge and the second column the target node.
The file is assumed to be sorted alphabetically.

The second data file `lawfareblog.csv.gz` contains the link structure for the lawfare blog.
Let's take a look at the first 10 of these lines:
```
$ zcat data/lawfareblog.csv.gz | head
source,target
www.lawfareblog.com/,www.lawfareblog.com/topic/interrogation
www.lawfareblog.com/,www.lawfareblog.com/upcoming-events
www.lawfareblog.com/,www.lawfareblog.com/
www.lawfareblog.com/,www.lawfareblog.com/our-comments-policy
www.lawfareblog.com/,www.lawfareblog.com/litigation-documents-related-appointment-matthew-whitaker-acting-attorney-general
www.lawfareblog.com/,www.lawfareblog.com/topic/lawfare-research-paper-series
www.lawfareblog.com/,www.lawfareblog.com/topic/book-reviews
www.lawfareblog.com/,www.lawfareblog.com/documents-related-mueller-investigation
www.lawfareblog.com/,www.lawfareblog.com/topic/international-law-loac
```
You can see that in this file, the node names are URLs.
Semantically, each line corresponds to an HTML `<a>` tag that is contained in the source webpage and links to the target webpage.

We can use the following command to count the total number of links in the file:
```
$ zcat data/lawfareblog.csv.gz | wc -l
1610789
```
Since every link corresponds to a non-zero entry in the `P` matrix,
this is also the value of `nnz(P)`.
(Technically, we should subtract 1 from this value since the `wc -l` command also counts the header line, not just the data lines.)

To get the dimensions of `P`, we need to count the total number of nodes in the graph.
The following command achieves this by: decompressing the file, extracting the first column, removing all duplicate lines, then counting the results.
```
$ zcat data/lawfareblog.csv.gz | cut -f1 -d, | uniq | wc -l
25761
```
This matrix is large enough that computing matrix products for dense matrices takes several minutes on a single CPU.
Fortunately, however, the matrix is very sparse.
The following python code computes the fraction of entries in the matrix with non-zero values:
```
>>> 1610788 / (25760**2)
0.0024274297384360172
```
Thus, by using sparse matrix operations, we will be able to speed up the code significantly.

**Code:**

The `pagerank.py` file contains code for loading the graph CSV files and searching through their nodes for key phrases.
For example, you can perform a search for all nodes (i.e. urls) that mention the string `corona` with the following command:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --search_query=corona
```

> **NOTE:**
> It will take about 10 seconds to load and parse the data files.
> All the other computation happens essentially instantly.

Currently, the pagerank of the nodes is not currently being calculated correctly, and so the webpages are returned in an arbitrary order.
Your task in this assignment will be to fix these calculations in order to have the most important results (i.e. highest pagerank results) returned first.

## Task 1: the power method

Implement the `WebGraph.power_method` function in `pagerank.py` for computing the pagerank vector by fixing the `FIXME` annotation.

**Part 1:**

To check that your implementation is working,
you should run the program on the `data/small.csv.gz` graph.
For my implementation, I get the following output.
```
$ python3 pagerank.py --data=data/small.csv.gz --verbose
DEBUG:root:computing indices
DEBUG:root:computing values
DEBUG:root:i=0 residual=2.5629e-01
DEBUG:root:i=1 residual=1.1841e-01
DEBUG:root:i=2 residual=7.0701e-02
DEBUG:root:i=3 residual=3.1815e-02
DEBUG:root:i=4 residual=2.0497e-02
DEBUG:root:i=5 residual=1.0108e-02
DEBUG:root:i=6 residual=6.3716e-03
DEBUG:root:i=7 residual=3.4228e-03
DEBUG:root:i=8 residual=2.0879e-03
DEBUG:root:i=9 residual=1.1750e-03
DEBUG:root:i=10 residual=7.0131e-04
DEBUG:root:i=11 residual=4.0321e-04
DEBUG:root:i=12 residual=2.3800e-04
DEBUG:root:i=13 residual=1.3812e-04
DEBUG:root:i=14 residual=8.1083e-05
DEBUG:root:i=15 residual=4.7251e-05
DEBUG:root:i=16 residual=2.7704e-05
DEBUG:root:i=17 residual=1.6164e-05
DEBUG:root:i=18 residual=9.4778e-06
DEBUG:root:i=19 residual=5.5066e-06
DEBUG:root:i=20 residual=3.2042e-06
DEBUG:root:i=21 residual=1.8612e-06
DEBUG:root:i=22 residual=1.1283e-06
DEBUG:root:i=23 residual=6.1907e-07
INFO:root:rank=0 pagerank=6.6270e-01 url=4
INFO:root:rank=1 pagerank=5.2179e-01 url=6
INFO:root:rank=2 pagerank=4.1434e-01 url=5
INFO:root:rank=3 pagerank=2.3175e-01 url=2
INFO:root:rank=4 pagerank=1.8590e-01 url=3
INFO:root:rank=5 pagerank=1.6917e-01 url=1
```
Yours likely won't be identical (due to weird floating point issues), but it should be similar.
In particular, the ranking of the nodes/urls should be the same order.

> **NOTE:**
> The `--verbose` flag causes all of the lines beginning with `DEBUG` to be printed.
> By default, only lines beginning with `INFO` are printed.

**Part 2:**

The `pagerank.py` file has an option `--search_query`, which takes a string as a parameter.
If this argument is used, then the program returns all nodes that match the query string sorted according to their pagerank.
Essentially, this gives us the most important pages related to our query.

Again, you may not get the exact same results as me,
but you should get similar results to the examples I've shown below.
Verify that you do in fact get similar results.

```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='corona'
INFO:root:rank=0 pagerank=1.0038e-03 url=www.lawfareblog.com/lawfare-podcast-united-nations-and-coronavirus-crisis
INFO:root:rank=1 pagerank=8.9224e-04 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
INFO:root:rank=2 pagerank=7.0390e-04 url=www.lawfareblog.com/britains-coronavirus-response
INFO:root:rank=3 pagerank=6.9153e-04 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
INFO:root:rank=4 pagerank=6.7041e-04 url=www.lawfareblog.com/israeli-emergency-regulations-location-tracking-coronavirus-carriers
INFO:root:rank=5 pagerank=6.6256e-04 url=www.lawfareblog.com/why-congress-conducting-business-usual-face-coronavirus
INFO:root:rank=6 pagerank=6.5046e-04 url=www.lawfareblog.com/congressional-homeland-security-committees-seek-ways-support-state-federal-responses-coronavirus
INFO:root:rank=7 pagerank=6.3620e-04 url=www.lawfareblog.com/paper-hearing-experts-debate-digital-contact-tracing-and-coronavirus-privacy-concerns
INFO:root:rank=8 pagerank=6.1248e-04 url=www.lawfareblog.com/house-subcommittee-voices-concerns-over-us-management-coronavirus
INFO:root:rank=9 pagerank=6.0187e-04 url=www.lawfareblog.com/livestream-house-oversight-committee-holds-hearing-government-coronavirus-response

$ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='trump'
INFO:root:rank=0 pagerank=5.7826e-03 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 pagerank=5.2338e-03 url=www.lawfareblog.com/document-trump-revokes-obama-executive-order-counterterrorism-strike-casualty-reporting
INFO:root:rank=2 pagerank=5.1297e-03 url=www.lawfareblog.com/trump-administrations-worrying-new-policy-israeli-settlements
INFO:root:rank=3 pagerank=4.6599e-03 url=www.lawfareblog.com/dc-circuit-overrules-district-courts-due-process-ruling-qasim-v-trump
INFO:root:rank=4 pagerank=4.5934e-03 url=www.lawfareblog.com/donald-trump-and-politically-weaponized-executive-branch
INFO:root:rank=5 pagerank=4.3071e-03 url=www.lawfareblog.com/how-trumps-approach-middle-east-ignores-past-future-and-human-condition
INFO:root:rank=6 pagerank=4.0935e-03 url=www.lawfareblog.com/why-trump-cant-buy-greenland
INFO:root:rank=7 pagerank=3.7591e-03 url=www.lawfareblog.com/oral-argument-summary-qassim-v-trump
INFO:root:rank=8 pagerank=3.4509e-03 url=www.lawfareblog.com/dc-circuit-court-denies-trump-rehearing-mazars-case
INFO:root:rank=9 pagerank=3.4484e-03 url=www.lawfareblog.com/second-circuit-rules-mazars-must-hand-over-trump-tax-returns-new-york-prosecutors

$ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='iran'
INFO:root:rank=0 pagerank=4.5746e-03 url=www.lawfareblog.com/praise-presidents-iran-tweets
INFO:root:rank=1 pagerank=4.4174e-03 url=www.lawfareblog.com/how-us-iran-tensions-could-disrupt-iraqs-fragile-peace
INFO:root:rank=2 pagerank=2.6928e-03 url=www.lawfareblog.com/cyber-command-operational-update-clarifying-june-2019-iran-operation
INFO:root:rank=3 pagerank=1.9391e-03 url=www.lawfareblog.com/aborted-iran-strike-fine-line-between-necessity-and-revenge
INFO:root:rank=4 pagerank=1.5452e-03 url=www.lawfareblog.com/parsing-state-departments-letter-use-force-against-iran
INFO:root:rank=5 pagerank=1.5357e-03 url=www.lawfareblog.com/iranian-hostage-crisis-and-its-effect-american-politics
INFO:root:rank=6 pagerank=1.5258e-03 url=www.lawfareblog.com/announcing-united-states-and-use-force-against-iran-new-lawfare-e-book
INFO:root:rank=7 pagerank=1.4221e-03 url=www.lawfareblog.com/us-names-iranian-revolutionary-guard-terrorist-organization-and-sanctions-international-criminal
INFO:root:rank=8 pagerank=1.1788e-03 url=www.lawfareblog.com/iran-shoots-down-us-drone-domestic-and-international-legal-implications
INFO:root:rank=9 pagerank=1.1463e-03 url=www.lawfareblog.com/israel-iran-syria-clash-and-law-use-force
```

**Part 3:**

The webgraph of lawfareblog.com (i.e. the `P` matrix) naturally contains a lot of structure.
For example, essentially all pages on the domain have links to the root page <https://lawfareblog.com/> and other "non-article" pages like <https://www.lawfareblog.com/topics> and <https://www.lawfareblog.com/subscribe-lawfare>.
These pages therefore have a large pagerank.
We can get a list of the pages with the largest pagerank by running

```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz
INFO:root:rank=0 pagerank=2.8741e-01 url=www.lawfareblog.com/lawfare-job-board
INFO:root:rank=1 pagerank=2.8741e-01 url=www.lawfareblog.com/masthead
INFO:root:rank=2 pagerank=2.8741e-01 url=www.lawfareblog.com/litigation-documents-related-appointment-matthew-whitaker-acting-attorney-general
INFO:root:rank=3 pagerank=2.8741e-01 url=www.lawfareblog.com/documents-related-mueller-investigation
INFO:root:rank=4 pagerank=2.8741e-01 url=www.lawfareblog.com/topics
INFO:root:rank=5 pagerank=2.8741e-01 url=www.lawfareblog.com/about-lawfare-brief-history-term-and-site
INFO:root:rank=6 pagerank=2.8741e-01 url=www.lawfareblog.com/snowden-revelations
INFO:root:rank=7 pagerank=2.8741e-01 url=www.lawfareblog.com/support-lawfare
INFO:root:rank=8 pagerank=2.8741e-01 url=www.lawfareblog.com/upcoming-events
INFO:root:rank=9 pagerank=2.8741e-01 url=www.lawfareblog.com/our-comments-policy
```

Most of these pages are not very interesting, however, because they are not articles,
and usually when we are performing a web search, we only want articles.

This raises the question: How can we find the most important articles filtering out the non-article pages?
The answer is to modify the `P` matrix by removing all links to non-article pages.

This raises another question: How do we know if a link is a non-article page?
Unfortunately, this is a hard question to answer with 100% accuracy,
but there are many methods that get us most of the way there.
One easy to implement method is to compute what's called the "in-link ratio" of each node (i.e. the total number of edges with the node as a target divided by the total number of nodes),
and then remove nodes from the search results with too-high of a ratio.
The intuition is that non-article pages often appear in the menu of a webpage, and so have links from almost all of the other webpages;
but article-webpages are unlikely to appear on a menu and so will only have a small number of links from other webpages.
The `--filter_ratio` parameter causes the code to remove all pages that have an in-link ratio larger than the provided value.

Using this option, we can estimate the most important articles on the domain with the following command:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2
INFO:root:rank=0 pagerank=3.4696e-01 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 pagerank=2.9521e-01 url=www.lawfareblog.com/livestream-nov-21-impeachment-hearings-0
INFO:root:rank=2 pagerank=2.9040e-01 url=www.lawfareblog.com/opening-statement-david-holmes
INFO:root:rank=3 pagerank=1.5179e-01 url=www.lawfareblog.com/lawfare-podcast-ben-nimmo-whack-mole-game-disinformation
INFO:root:rank=4 pagerank=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1963
INFO:root:rank=5 pagerank=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1964
INFO:root:rank=6 pagerank=1.5071e-01 url=www.lawfareblog.com/lawfare-podcast-week-was-impeachment
INFO:root:rank=7 pagerank=1.4957e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1962
INFO:root:rank=8 pagerank=1.4367e-01 url=www.lawfareblog.com/cyberlaw-podcast-mistrusting-google
INFO:root:rank=9 pagerank=1.4240e-01 url=www.lawfareblog.com/lawfare-podcast-bonus-edition-gordon-sondland-vs-committee-no-bull
```
Notice that the urls in this list look much more like articles than the urls in the previous list.

When Google calculates their `P` matrix for the web,
they use a similar (but much more complicated) process to modify the `P` matrix in order to reduce spam results.
The exact formula they use is a jealously guarded secret that they update continuously.

In the case above, notice that we have accidentally removed the blog's most popular article (<www.lawfareblog.com/snowden-revelations>).
The blog editors believed that Snowden's revelations about NSA spying are so important that they directly put a link to the article on the menu.
So every single webpage in the domain links to the Snowden article,
and our "anti-spam" `--filter-ratio` argument removed this article from the list.
In general, it is a challenging open problem to remove spam from pagerank results,
and all current solutions rely on careful human tuning and still have lots of false positives and false negatives.

**Part 4:**

Recall from the reading that the runtime of pagerank depends heavily on the eigengap of the `\bar\bar P` matrix,
and that this eigengap is bounded by the alpha parameter.

Run the following four commands:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose 
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --alpha=0.99999
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2 --alpha=0.99999
```
You should notice that the last command takes considerably more iterations to compute the pagerank vector.
(My code takes 685 iterations for this call, and about 10 iterations for all the others.)

This raises the question: Why does the second command (with the `--alpha` option but without the `--filter_ratio`) option not take a long time to run?
The answer is that the `P` graph for <https://www.lawfareblog.com> naturally has a large eigengap and so is fast to compute for all alpha values,
but the modified graph does not have a large eigengap and so requires a small alpha for fast convergence.

Changing the value of alpha also gives us very different pagerank rankings.
For example, 
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2
INFO:root:rank=0 pagerank=3.4696e-01 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 pagerank=2.9521e-01 url=www.lawfareblog.com/livestream-nov-21-impeachment-hearings-0
INFO:root:rank=2 pagerank=2.9040e-01 url=www.lawfareblog.com/opening-statement-david-holmes
INFO:root:rank=3 pagerank=1.5179e-01 url=www.lawfareblog.com/lawfare-podcast-ben-nimmo-whack-mole-game-disinformation
INFO:root:rank=4 pagerank=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1963
INFO:root:rank=5 pagerank=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1964
INFO:root:rank=6 pagerank=1.5071e-01 url=www.lawfareblog.com/lawfare-podcast-week-was-impeachment
INFO:root:rank=7 pagerank=1.4957e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1962
INFO:root:rank=8 pagerank=1.4367e-01 url=www.lawfareblog.com/cyberlaw-podcast-mistrusting-google
INFO:root:rank=9 pagerank=1.4240e-01 url=www.lawfareblog.com/lawfare-podcast-bonus-edition-gordon-sondland-vs-committee-no-bull

$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2 --alpha=0.99999
INFO:root:rank=0 pagerank=7.0149e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 pagerank=7.0149e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 pagerank=1.0552e-01 url=www.lawfareblog.com/cost-using-zero-days
INFO:root:rank=3 pagerank=3.1755e-02 url=www.lawfareblog.com/lawfare-podcast-former-congressman-brian-baird-and-daniel-schuman-how-congress-can-continue-function
INFO:root:rank=4 pagerank=2.2040e-02 url=www.lawfareblog.com/events
INFO:root:rank=5 pagerank=1.6027e-02 url=www.lawfareblog.com/water-wars-increased-us-focus-indo-pacific
INFO:root:rank=6 pagerank=1.6026e-02 url=www.lawfareblog.com/water-wars-drill-maybe-drill
INFO:root:rank=7 pagerank=1.6023e-02 url=www.lawfareblog.com/water-wars-disjointed-operations-south-china-sea
INFO:root:rank=8 pagerank=1.6020e-02 url=www.lawfareblog.com/water-wars-song-oil-and-fire
INFO:root:rank=9 pagerank=1.6020e-02 url=www.lawfareblog.com/water-wars-sinking-feeling-philippine-china-relations
```

Which of these rankings is better is entirely subjective,
and the only way to know if you have the "best" alpha for your application is to try several variations and see what is best.
If large alphas are good for your application, you can see that there is a trade-off between quality answers and algorithmic runtime.
We'll be exploring this trade-off more formally in class over the rest of the semester.

## Task 2: the personalization vector

The most interesting applications of pagerank involve the personalization vector.
Implement the `WebGraph.make_personalization_vector` function so that it outputs a personalization vector tuned for the input query.
The pseudocode for the function is:
```
for each index in the personalization vector:
    get the url for the index (see the _index_to_url function)
    check if the url satisfies the input query (see the url_satisfies_query function)
    if so, set the corresponding index to one
normalize the vector
```

**Part 1:**

The command line argument `--personalization_vector_query` will use the function you created above to augment your search with a custom personalization vector.
If you've implemented the function correctly,
you should get results similar to:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --personalization_vector_query='corona'
INFO:root:rank=0 pagerank=6.3127e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 pagerank=6.3124e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 pagerank=1.5947e-01 url=www.lawfareblog.com/chinatalk-how-party-takes-its-propaganda-global
INFO:root:rank=3 pagerank=1.2209e-01 url=www.lawfareblog.com/brexit-not-immune-coronavirus
INFO:root:rank=4 pagerank=1.2209e-01 url=www.lawfareblog.com/rational-security-my-corona-edition
INFO:root:rank=5 pagerank=9.3360e-02 url=www.lawfareblog.com/trump-cant-reopen-country-over-state-objections
INFO:root:rank=6 pagerank=9.1920e-02 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
INFO:root:rank=7 pagerank=9.1920e-02 url=www.lawfareblog.com/britains-coronavirus-response
INFO:root:rank=8 pagerank=7.7770e-02 url=www.lawfareblog.com/lawfare-podcast-united-nations-and-coronavirus-crisis
INFO:root:rank=9 pagerank=7.2888e-02 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
```

Notice that these results are significantly different than when using the `--search_query` option:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --search_query='corona'
INFO:root:rank=0 pagerank=8.1320e-03 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
INFO:root:rank=1 pagerank=7.7908e-03 url=www.lawfareblog.com/lawfare-podcast-united-nations-and-coronavirus-crisis
INFO:root:rank=2 pagerank=5.2262e-03 url=www.lawfareblog.com/livestream-house-oversight-committee-holds-hearing-government-coronavirus-response
INFO:root:rank=3 pagerank=3.9584e-03 url=www.lawfareblog.com/britains-coronavirus-response
INFO:root:rank=4 pagerank=3.8114e-03 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
INFO:root:rank=5 pagerank=3.3973e-03 url=www.lawfareblog.com/paper-hearing-experts-debate-digital-contact-tracing-and-coronavirus-privacy-concerns
INFO:root:rank=6 pagerank=3.3633e-03 url=www.lawfareblog.com/cyberlaw-podcast-how-israel-fighting-coronavirus
INFO:root:rank=7 pagerank=3.3557e-03 url=www.lawfareblog.com/israeli-emergency-regulations-location-tracking-coronavirus-carriers
INFO:root:rank=8 pagerank=3.2160e-03 url=www.lawfareblog.com/congress-needs-coronavirus-failsafe-its-too-late
INFO:root:rank=9 pagerank=3.1036e-03 url=www.lawfareblog.com/why-congress-conducting-business-usual-face-coronavirus
```

Which results are better?
Again, that depends on what you mean by "better."
With the `--personalization_vector_query` option,
a webpage is important only if other coronavirus webpages also think it's important;
with the `--search_query` option,
a webpage is important if any other webpage thinks it's important.
You'll notice that in the later example, many of the webpages are about Congressional proceedings related to the coronavirus.
From a strictly coronavirus perspective, these are not very important webpages.
But in the broader context of national security, these are very important webpages.

Google engineers spend TONs of time fine-tuning their pagerank personalization vectors to remove spam webpages.
Exactly how they do this is another one of their secrets that they don't publicly talk about.

**Part 2:**

Another use of the `--personalization_vector_query` option is that we can find out what webpages are related to the coronavirus but don't directly mention the coronavirus.
This can be used to map out what types of topics are similar to the coronavirus.

For example, the following query ranks all webpages by their `corona` importance,
but removes webpages mentioning `corona` from the results.
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --personalization_vector_query='corona' --search_query='-corona'
INFO:root:rank=0 pagerank=6.3127e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 pagerank=6.3124e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 pagerank=1.5947e-01 url=www.lawfareblog.com/chinatalk-how-party-takes-its-propaganda-global
INFO:root:rank=3 pagerank=9.3360e-02 url=www.lawfareblog.com/trump-cant-reopen-country-over-state-objections
INFO:root:rank=4 pagerank=7.0277e-02 url=www.lawfareblog.com/fault-lines-foreign-policy-quarantined
INFO:root:rank=5 pagerank=6.9713e-02 url=www.lawfareblog.com/lawfare-podcast-mom-and-dad-talk-clinical-trials-pandemic
INFO:root:rank=6 pagerank=6.4944e-02 url=www.lawfareblog.com/limits-world-health-organization
INFO:root:rank=7 pagerank=5.9492e-02 url=www.lawfareblog.com/chinatalk-dispatches-shanghai-beijing-and-hong-kong
INFO:root:rank=8 pagerank=5.1245e-02 url=www.lawfareblog.com/us-moves-dismiss-case-against-company-linked-ira-troll-farm
INFO:root:rank=9 pagerank=5.1245e-02 url=www.lawfareblog.com/livestream-house-armed-services-holds-hearing-national-security-challenges-north-and-south-america
```
You can see that there are many urls about concepts that are obviously related like "covid", "clinical trials", and "quarantine",
but this algorithm also finds articles about Chinese propaganda and Trump's policy decisions.
Both of these articles are highly relevant to coronavirus discussions,
but a simple keyword search for corona or related terms would not find these articles.

<!--
**Part 3:**

Select another topic related to national security.
You should experiment with a national security topic other than the coronavirus.
For example, find out what articles are important to the `iran` topic but do not contain the word `iran`.
Your goal should be to discover what topics that www.lawfareblog.com considers to be related to the national security topic you choose.
-->

## Submission

1. Create a new repo on github (not a fork of this repo).

1. Run the following commands, and paste their output into the code blocks below.
   
   Task 1, part 1:
   ```
   $ python3 pagerank.py --data=data/small.csv.gz --verbose
   DEBUG:root:computing indices
DEBUG:root:computing values
DEBUG:root:i=0 residual=0.2562914192676544
DEBUG:root:i=1 residual=0.11841227114200592
DEBUG:root:i=2 residual=0.07070134580135345
DEBUG:root:i=3 residual=0.03181542828679085
DEBUG:root:i=4 residual=0.020496590062975883
DEBUG:root:i=5 residual=0.01010835450142622
DEBUG:root:i=6 residual=0.006371544674038887
DEBUG:root:i=7 residual=0.0034227892756462097
DEBUG:root:i=8 residual=0.002087961183860898
DEBUG:root:i=9 residual=0.0011749734403565526
DEBUG:root:i=10 residual=0.0007012754795141518
DEBUG:root:i=11 residual=0.00040320929838344455
DEBUG:root:i=12 residual=0.00023798426263965666
DEBUG:root:i=13 residual=0.00013812065299134701
DEBUG:root:i=14 residual=8.108324982458726e-05
DEBUG:root:i=15 residual=4.7269360948121175e-05
DEBUG:root:i=16 residual=2.7704918466042727e-05
DEBUG:root:i=17 residual=1.6170568414963782e-05
DEBUG:root:i=18 residual=9.479118489252869e-06
DEBUG:root:i=19 residual=5.4782999541203026e-06
DEBUG:root:i=20 residual=3.2123323308042018e-06
DEBUG:root:i=21 residual=1.8802053318722756e-06
DEBUG:root:i=22 residual=1.1228398761886638e-06
DEBUG:root:i=23 residual=6.322027275018627e-07
INFO:root:rank=0 pagerank=6.6270e-01 url=4
INFO:root:rank=1 pagerank=5.2179e-01 url=6
INFO:root:rank=2 pagerank=4.1434e-01 url=5
INFO:root:rank=3 pagerank=2.3175e-01 url=2
INFO:root:rank=4 pagerank=1.8590e-01 url=3
INFO:root:rank=5 pagerank=1.6917e-01 url=1
   ```

   Task 1, part 2:
   ```
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='corona'
   python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='corona'INFO:root:rank=0 pagerank=1.0038e-03 url=www.lawfareblog.com/lawfare-podcast-united-nations-and-coronavirus-crisis
INFO:root:rank=1 pagerank=8.9224e-04 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
INFO:root:rank=2 pagerank=7.0390e-04 url=www.lawfareblog.com/britains-coronavirus-response
INFO:root:rank=3 pagerank=6.9153e-04 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
INFO:root:rank=4 pagerank=6.7041e-04 url=www.lawfareblog.com/israeli-emergency-regulations-location-tracking-coronavirus-carriers
INFO:root:rank=5 pagerank=6.6256e-04 url=www.lawfareblog.com/why-congress-conducting-business-usual-face-coronavirus
INFO:root:rank=6 pagerank=6.5046e-04 url=www.lawfareblog.com/congressional-homeland-security-committees-seek-ways-support-state-federal-responses-coronavirus
INFO:root:rank=7 pagerank=6.3620e-04 url=www.lawfareblog.com/paper-hearing-experts-debate-digital-contact-tracing-and-coronavirus-privacy-concerns
INFO:root:rank=8 pagerank=6.1248e-04 url=www.lawfareblog.com/house-subcommittee-voices-concerns-over-us-management-coronavirus
INFO:root:rank=9 pagerank=6.0187e-04 url=www.lawfareblog.com/livestream-house-oversight-committee-holds-hearing-government-coronavirus-response

   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='trump'
   INFO:root:rank=0 pagerank=5.7826e-03 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 pagerank=5.2338e-03 url=www.lawfareblog.com/document-trump-revokes-obama-executive-order-counterterrorism-strike-casualty-reporting
INFO:root:rank=2 pagerank=5.1297e-03 url=www.lawfareblog.com/trump-administrations-worrying-new-policy-israeli-settlements
INFO:root:rank=3 pagerank=4.6599e-03 url=www.lawfareblog.com/dc-circuit-overrules-district-courts-due-process-ruling-qasim-v-trump
INFO:root:rank=4 pagerank=4.5934e-03 url=www.lawfareblog.com/donald-trump-and-politically-weaponized-executive-branch
INFO:root:rank=5 pagerank=4.3071e-03 url=www.lawfareblog.com/how-trumps-approach-middle-east-ignores-past-future-and-human-condition
INFO:root:rank=6 pagerank=4.0935e-03 url=www.lawfareblog.com/why-trump-cant-buy-greenland
INFO:root:rank=7 pagerank=3.7591e-03 url=www.lawfareblog.com/oral-argument-summary-qassim-v-trump
INFO:root:rank=8 pagerank=3.4509e-03 url=www.lawfareblog.com/dc-circuit-court-denies-trump-rehearing-mazars-case
INFO:root:rank=9 pagerank=3.4484e-03 url=www.lawfareblog.com/second-circuit-rules-mazars-must-hand-over-trump-tax-returns-new-york-prosecutors

   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='iran'
   INFO:root:rank=0 pagerank=4.5746e-03 url=www.lawfareblog.com/praise-presidents-iran-tweets
INFO:root:rank=1 pagerank=4.4174e-03 url=www.lawfareblog.com/how-us-iran-tensions-could-disrupt-iraqs-fragile-peace
INFO:root:rank=2 pagerank=2.6928e-03 url=www.lawfareblog.com/cyber-command-operational-update-clarifying-june-2019-iran-operation
INFO:root:rank=3 pagerank=1.9391e-03 url=www.lawfareblog.com/aborted-iran-strike-fine-line-between-necessity-and-revenge
INFO:root:rank=4 pagerank=1.5452e-03 url=www.lawfareblog.com/parsing-state-departments-letter-use-force-against-iran
INFO:root:rank=5 pagerank=1.5357e-03 url=www.lawfareblog.com/iranian-hostage-crisis-and-its-effect-american-politics
INFO:root:rank=6 pagerank=1.5258e-03 url=www.lawfareblog.com/announcing-united-states-and-use-force-against-iran-new-lawfare-e-book
INFO:root:rank=7 pagerank=1.4221e-03 url=www.lawfareblog.com/us-names-iranian-revolutionary-guard-terrorist-organization-and-sanctions-international-criminal
INFO:root:rank=8 pagerank=1.1788e-03 url=www.lawfareblog.com/iran-shoots-down-us-drone-domestic-and-international-legal-implications
INFO:root:rank=9 pagerank=1.1463e-03 url=www.lawfareblog.com/israel-iran-syria-clash-and-law-use-force
   ```

   Task 1, part 3:
   ```
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz
   INFO:root:rank=0 pagerank=2.8741e-01 url=www.lawfareblog.com/lawfare-job-board
INFO:root:rank=1 pagerank=2.8741e-01 url=www.lawfareblog.com/masthead
INFO:root:rank=2 pagerank=2.8741e-01 url=www.lawfareblog.com/litigation-documents-related-appointment-matthew-whitaker-acting-attorney-general
INFO:root:rank=3 pagerank=2.8741e-01 url=www.lawfareblog.com/documents-related-mueller-investigation
INFO:root:rank=4 pagerank=2.8741e-01 url=www.lawfareblog.com/topics
INFO:root:rank=5 pagerank=2.8741e-01 url=www.lawfareblog.com/about-lawfare-brief-history-term-and-site
INFO:root:rank=6 pagerank=2.8741e-01 url=www.lawfareblog.com/snowden-revelations
INFO:root:rank=7 pagerank=2.8741e-01 url=www.lawfareblog.com/support-lawfare
INFO:root:rank=8 pagerank=2.8741e-01 url=www.lawfareblog.com/upcoming-events
INFO:root:rank=9 pagerank=2.8741e-01 url=www.lawfareblog.com/our-comments-policy


   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2
   INFO:root:rank=0 pagerank=3.4696e-01 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 pagerank=2.9521e-01 url=www.lawfareblog.com/livestream-nov-21-impeachment-hearings-0
INFO:root:rank=2 pagerank=2.9040e-01 url=www.lawfareblog.com/opening-statement-david-holmes
INFO:root:rank=3 pagerank=1.5179e-01 url=www.lawfareblog.com/lawfare-podcast-ben-nimmo-whack-mole-game-disinformation
INFO:root:rank=4 pagerank=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1963
INFO:root:rank=5 pagerank=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1964
INFO:root:rank=6 pagerank=1.5071e-01 url=www.lawfareblog.com/lawfare-podcast-week-was-impeachment
INFO:root:rank=7 pagerank=1.4957e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1962
INFO:root:rank=8 pagerank=1.4367e-01 url=www.lawfareblog.com/cyberlaw-podcast-mistrusting-google
INFO:root:rank=9 pagerank=1.4240e-01 url=www.lawfareblog.com/lawfare-podcast-bonus-edition-gordon-sondland-vs-committee-no-bull
   ```

   Task 1, part 4:
   ```
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose
   DEBUG:root:computing indices
DEBUG:root:computing values
DEBUG:root:i=0 residual=1.3793821334838867
DEBUG:root:i=1 residual=0.11642514914274216
DEBUG:root:i=2 residual=0.07495073974132538
DEBUG:root:i=3 residual=0.031712669879198074
DEBUG:root:i=4 residual=0.01746140420436859
DEBUG:root:i=5 residual=0.008529536426067352
DEBUG:root:i=6 residual=0.0044392067939043045
DEBUG:root:i=7 residual=0.002238823566585779
DEBUG:root:i=8 residual=0.0011464565759524703
DEBUG:root:i=9 residual=0.0005798051133751869
DEBUG:root:i=10 residual=0.00029213540256023407
DEBUG:root:i=11 residual=0.00014553092478308827
DEBUG:root:i=12 residual=7.149828888941556e-05
DEBUG:root:i=13 residual=3.433692472754046e-05
DEBUG:root:i=14 residual=1.5638515833416022e-05
DEBUG:root:i=15 residual=6.266389391385019e-06
DEBUG:root:i=16 residual=2.8251236017240444e-06
DEBUG:root:i=17 residual=1.3609912912215805e-06
DEBUG:root:i=18 residual=4.311152963509812e-07
INFO:root:rank=0 pagerank=2.8741e-01 url=www.lawfareblog.com/lawfare-job-board
INFO:root:rank=1 pagerank=2.8741e-01 url=www.lawfareblog.com/masthead
INFO:root:rank=2 pagerank=2.8741e-01 url=www.lawfareblog.com/litigation-documents-related-appointment-matthew-whitaker-acting-attorney-general
INFO:root:rank=3 pagerank=2.8741e-01 url=www.lawfareblog.com/documents-related-mueller-investigation
INFO:root:rank=4 pagerank=2.8741e-01 url=www.lawfareblog.com/topics
INFO:root:rank=5 pagerank=2.8741e-01 url=www.lawfareblog.com/about-lawfare-brief-history-term-and-site
INFO:root:rank=6 pagerank=2.8741e-01 url=www.lawfareblog.com/snowden-revelations
INFO:root:rank=7 pagerank=2.8741e-01 url=www.lawfareblog.com/support-lawfare
INFO:root:rank=8 pagerank=2.8741e-01 url=www.lawfareblog.com/upcoming-events
INFO:root:rank=9 pagerank=2.8741e-01 url=www.lawfareblog.com/our-comments-policy
Samuel.Chotzen.23@lambda-server:~/Chotzen-Pagerank-Project$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --alpha=0.99999
DEBUG:root:computing indices
DEBUG:root:computing values
DEBUG:root:i=0 residual=1.3845853805541992
DEBUG:root:i=1 residual=0.07087279111146927
DEBUG:root:i=2 residual=0.01881912723183632
DEBUG:root:i=3 residual=0.006956098601222038
DEBUG:root:i=4 residual=0.0027344103436917067
DEBUG:root:i=5 residual=0.0010338517604395747
DEBUG:root:i=6 residual=0.0003771328483708203
DEBUG:root:i=7 residual=0.00013517338084056973
DEBUG:root:i=8 residual=4.816377622773871e-05
DEBUG:root:i=9 residual=1.7143036529887468e-05
DEBUG:root:i=10 residual=6.101055532781174e-06
DEBUG:root:i=11 residual=2.172899030483677e-06
DEBUG:root:i=12 residual=7.789812457303924e-07
INFO:root:rank=0 pagerank=2.8859e-01 url=www.lawfareblog.com/lawfare-job-board
INFO:root:rank=1 pagerank=2.8859e-01 url=www.lawfareblog.com/masthead
INFO:root:rank=2 pagerank=2.8859e-01 url=www.lawfareblog.com/litigation-documents-related-appointment-matthew-whitaker-acting-attorney-general
INFO:root:rank=3 pagerank=2.8859e-01 url=www.lawfareblog.com/documents-related-mueller-investigation
INFO:root:rank=4 pagerank=2.8859e-01 url=www.lawfareblog.com/topics
INFO:root:rank=5 pagerank=2.8859e-01 url=www.lawfareblog.com/about-lawfare-brief-history-term-and-site
INFO:root:rank=6 pagerank=2.8859e-01 url=www.lawfareblog.com/snowden-revelations
INFO:root:rank=7 pagerank=2.8859e-01 url=www.lawfareblog.com/support-lawfare
INFO:root:rank=8 pagerank=2.8859e-01 url=www.lawfareblog.com/upcoming-events
INFO:root:rank=9 pagerank=2.8859e-01 url=www.lawfareblog.com/our-comments-policy
   
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --alpha=0.99999
   DEBUG:root:computing indices
DEBUG:root:computing values
DEBUG:root:i=0 residual=1.3845853805541992
DEBUG:root:i=1 residual=0.07087279111146927
DEBUG:root:i=2 residual=0.01881912723183632
DEBUG:root:i=3 residual=0.006956098601222038
DEBUG:root:i=4 residual=0.0027344103436917067
DEBUG:root:i=5 residual=0.0010338517604395747
DEBUG:root:i=6 residual=0.0003771328483708203
DEBUG:root:i=7 residual=0.00013517338084056973
DEBUG:root:i=8 residual=4.816377622773871e-05
DEBUG:root:i=9 residual=1.7143036529887468e-05
DEBUG:root:i=10 residual=6.101055532781174e-06
DEBUG:root:i=11 residual=2.172899030483677e-06
DEBUG:root:i=12 residual=7.789812457303924e-07
INFO:root:rank=0 pagerank=2.8859e-01 url=www.lawfareblog.com/lawfare-job-board
INFO:root:rank=1 pagerank=2.8859e-01 url=www.lawfareblog.com/masthead
INFO:root:rank=2 pagerank=2.8859e-01 url=www.lawfareblog.com/litigation-documents-related-appointment-matthew-whitaker-acting-attorney-general
INFO:root:rank=3 pagerank=2.8859e-01 url=www.lawfareblog.com/documents-related-mueller-investigation
INFO:root:rank=4 pagerank=2.8859e-01 url=www.lawfareblog.com/topics
INFO:root:rank=5 pagerank=2.8859e-01 url=www.lawfareblog.com/about-lawfare-brief-history-term-and-site
INFO:root:rank=6 pagerank=2.8859e-01 url=www.lawfareblog.com/snowden-revelations
INFO:root:rank=7 pagerank=2.8859e-01 url=www.lawfareblog.com/support-lawfare
INFO:root:rank=8 pagerank=2.8859e-01 url=www.lawfareblog.com/upcoming-events
INFO:root:rank=9 pagerank=2.8859e-01 url=www.lawfareblog.com/our-comments-policy

   
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2
   DEBUG:root:computing indices
DEBUG:root:computing values
DEBUG:root:i=0 residual=1.2609827518463135
DEBUG:root:i=1 residual=0.49858558177948
DEBUG:root:i=2 residual=0.13420072197914124
DEBUG:root:i=3 residual=0.0692318007349968
DEBUG:root:i=4 residual=0.023411516100168228
DEBUG:root:i=5 residual=0.010188630782067776
DEBUG:root:i=6 residual=0.004910613875836134
DEBUG:root:i=7 residual=0.002279882086440921
DEBUG:root:i=8 residual=0.0010739191202446818
DEBUG:root:i=9 residual=0.0005249588284641504
DEBUG:root:i=10 residual=0.00026969940518029034
DEBUG:root:i=11 residual=0.00014575093518942595
DEBUG:root:i=12 residual=8.241771865868941e-05
DEBUG:root:i=13 residual=4.819605601369403e-05
DEBUG:root:i=14 residual=2.881278851418756e-05
DEBUG:root:i=15 residual=1.7407368432031944e-05
DEBUG:root:i=16 residual=1.0556699635344557e-05
DEBUG:root:i=17 residual=6.386945642589126e-06
DEBUG:root:i=18 residual=3.835165898635751e-06
DEBUG:root:i=19 residual=2.295828835485736e-06
DEBUG:root:i=20 residual=1.3655146631208481e-06
DEBUG:root:i=21 residual=8.102273341137334e-07
INFO:root:rank=0 pagerank=3.4696e-01 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 pagerank=2.9521e-01 url=www.lawfareblog.com/livestream-nov-21-impeachment-hearings-0
INFO:root:rank=2 pagerank=2.9040e-01 url=www.lawfareblog.com/opening-statement-david-holmes
INFO:root:rank=3 pagerank=1.5179e-01 url=www.lawfareblog.com/lawfare-podcast-ben-nimmo-whack-mole-game-disinformation
INFO:root:rank=4 pagerank=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1963
INFO:root:rank=5 pagerank=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1964
INFO:root:rank=6 pagerank=1.5071e-01 url=www.lawfareblog.com/lawfare-podcast-week-was-impeachment
INFO:root:rank=7 pagerank=1.4957e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1962
INFO:root:rank=8 pagerank=1.4367e-01 url=www.lawfareblog.com/cyberlaw-podcast-mistrusting-google
INFO:root:rank=9 pagerank=1.4240e-01 url=www.lawfareblog.com/lawfare-podcast-bonus-edition-gordon-sondland-vs-committee-no-bull
   
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2 --alpha=0.99999
   DEBUG:root:computing indices
DEBUG:root:computing values
DEBUG:root:i=0 residual=1.2808992862701416
DEBUG:root:i=1 residual=0.5632966160774231
DEBUG:root:i=2 residual=0.38343238830566406
DEBUG:root:i=3 residual=0.21868214011192322
DEBUG:root:i=4 residual=0.14109930396080017
DEBUG:root:i=5 residual=0.10877226293087006
DEBUG:root:i=6 residual=0.09297246485948563
DEBUG:root:i=7 residual=0.08236386626958847
DEBUG:root:i=8 residual=0.07349604368209839
DEBUG:root:i=9 residual=0.0657142698764801
DEBUG:root:i=10 residual=0.05918187275528908
DEBUG:root:i=11 residual=0.054233964532613754
DEBUG:root:i=12 residual=0.05114196613430977
DEBUG:root:i=13 residual=0.049983780831098557
DEBUG:root:i=14 residual=0.05056799203157425
DEBUG:root:i=15 residual=0.05246277526021004
DEBUG:root:i=16 residual=0.055112894624471664
DEBUG:root:i=17 residual=0.057960107922554016
DEBUG:root:i=18 residual=0.060519684106111526
DEBUG:root:i=19 residual=0.06241811439394951
DEBUG:root:i=20 residual=0.0634097307920456
DEBUG:root:i=21 residual=0.06338082998991013
DEBUG:root:i=22 residual=0.062340330332517624
DEBUG:root:i=23 residual=0.060396045446395874
DEBUG:root:i=24 residual=0.05772068351507187
DEBUG:root:i=25 residual=0.05451760068535805
DEBUG:root:i=26 residual=0.05098811537027359
DEBUG:root:i=27 residual=0.04731093719601631
DEBUG:root:i=28 residual=0.04363013803958893
DEBUG:root:i=29 residual=0.04005275294184685
DEBUG:root:i=30 residual=0.036651611328125
DEBUG:root:i=31 residual=0.03347074240446091
DEBUG:root:i=32 residual=0.030532410368323326
DEBUG:root:i=33 residual=0.027842361479997635
DEBUG:root:i=34 residual=0.025396058335900307
DEBUG:root:i=35 residual=0.02318168617784977
DEBUG:root:i=36 residual=0.021183349192142487
DEBUG:root:i=37 residual=0.019383760169148445
DEBUG:root:i=38 residual=0.017764851450920105
DEBUG:root:i=39 residual=0.01630914956331253
DEBUG:root:i=40 residual=0.014999964274466038
DEBUG:root:i=41 residual=0.01382177509367466
DEBUG:root:i=42 residual=0.012760654091835022
DEBUG:root:i=43 residual=0.011803763918578625
DEBUG:root:i=44 residual=0.010939867235720158
DEBUG:root:i=45 residual=0.010158696211874485
DEBUG:root:i=46 residual=0.009451234713196754
DEBUG:root:i=47 residual=0.008809519931674004
DEBUG:root:i=48 residual=0.008226451463997364
DEBUG:root:i=49 residual=0.007695693057030439
DEBUG:root:i=50 residual=0.007211844902485609
DEBUG:root:i=51 residual=0.006769946310669184
DEBUG:root:i=52 residual=0.00636557349935174
DEBUG:root:i=53 residual=0.005995068699121475
DEBUG:root:i=54 residual=0.005654900800436735
DEBUG:root:i=55 residual=0.005342118442058563
DEBUG:root:i=56 residual=0.005054069682955742
DEBUG:root:i=57 residual=0.0047883219085633755
DEBUG:root:i=58 residual=0.00454278290271759
DEBUG:root:i=59 residual=0.004315555095672607
DEBUG:root:i=60 residual=0.004104920197278261
DEBUG:root:i=61 residual=0.0039094192907214165
DEBUG:root:i=62 residual=0.0037276900839060545
DEBUG:root:i=63 residual=0.0035584864672273397
DEBUG:root:i=64 residual=0.00340078491717577
DEBUG:root:i=65 residual=0.0032535435166209936
DEBUG:root:i=66 residual=0.003115894738584757
DEBUG:root:i=67 residual=0.0029870672151446342
DEBUG:root:i=68 residual=0.002866321476176381
DEBUG:root:i=69 residual=0.0027530149091035128
DEBUG:root:i=70 residual=0.0026465884875506163
DEBUG:root:i=71 residual=0.00254644057713449
DEBUG:root:i=72 residual=0.0024521583691239357
DEBUG:root:i=73 residual=0.002363257808610797
DEBUG:root:i=74 residual=0.002279372187331319
DEBUG:root:i=75 residual=0.002200088230893016
DEBUG:root:i=76 residual=0.0021251074504107237
DEBUG:root:i=77 residual=0.002054116688668728
DEBUG:root:i=78 residual=0.001986836316064
DEBUG:root:i=79 residual=0.0019230113830417395
DEBUG:root:i=80 residual=0.0018623936921358109
DEBUG:root:i=81 residual=0.0018048037309199572
DEBUG:root:i=82 residual=0.0017499898094683886
DEBUG:root:i=83 residual=0.0016978095518425107
DEBUG:root:i=84 residual=0.0016480942722409964
DEBUG:root:i=85 residual=0.0016006676014512777
DEBUG:root:i=86 residual=0.0015554048586636782
DEBUG:root:i=87 residual=0.0015121562173590064
DEBUG:root:i=88 residual=0.0014708094531670213
DEBUG:root:i=89 residual=0.0014312585117295384
DEBUG:root:i=90 residual=0.001393393729813397
DEBUG:root:i=91 residual=0.0013571084709838033
DEBUG:root:i=92 residual=0.0013223018031567335
DEBUG:root:i=93 residual=0.001288916333578527
DEBUG:root:i=94 residual=0.0012568600941449404
DEBUG:root:i=95 residual=0.001226069056428969
DEBUG:root:i=96 residual=0.0011964471777901053
DEBUG:root:i=97 residual=0.0011679715244099498
DEBUG:root:i=98 residual=0.0011405546683818102
DEBUG:root:i=99 residual=0.0011141335126012564
DEBUG:root:i=100 residual=0.001088679302483797
DEBUG:root:i=101 residual=0.0010641359258443117
DEBUG:root:i=102 residual=0.0010404664790257812
DEBUG:root:i=103 residual=0.0010176090290769935
DEBUG:root:i=104 residual=0.000995548558421433
DEBUG:root:i=105 residual=0.0009742120746523142
DEBUG:root:i=106 residual=0.0009535759454593062
DEBUG:root:i=107 residual=0.0009336327784694731
DEBUG:root:i=108 residual=0.0009143375209532678
DEBUG:root:i=109 residual=0.0008956270758062601
DEBUG:root:i=110 residual=0.0008775161113590002
DEBUG:root:i=111 residual=0.0008599707507528365
DEBUG:root:i=112 residual=0.0008429399458691478
DEBUG:root:i=113 residual=0.0008264374337159097
DEBUG:root:i=114 residual=0.0008104239241220057
DEBUG:root:i=115 residual=0.0007948762504383922
DEBUG:root:i=116 residual=0.0007797730504535139
DEBUG:root:i=117 residual=0.0007651032647117972
DEBUG:root:i=118 residual=0.0007508292910642922
DEBUG:root:i=119 residual=0.0007369727827608585
DEBUG:root:i=120 residual=0.000723482808098197
DEBUG:root:i=121 residual=0.0007103553507477045
DEBUG:root:i=122 residual=0.0006975746364332736
DEBUG:root:i=123 residual=0.0006851448561064899
DEBUG:root:i=124 residual=0.0006730218883603811
DEBUG:root:i=125 residual=0.0006612307042814791
DEBUG:root:i=126 residual=0.0006497257272712886
DEBUG:root:i=127 residual=0.0006385008455254138
DEBUG:root:i=128 residual=0.0006275647901929915
DEBUG:root:i=129 residual=0.0006168966647237539
DEBUG:root:i=130 residual=0.0006064838380552828
DEBUG:root:i=131 residual=0.0005963284056633711
DEBUG:root:i=132 residual=0.0005864074919372797
DEBUG:root:i=133 residual=0.0005767039838247001
DEBUG:root:i=134 residual=0.0005672496045008302
DEBUG:root:i=135 residual=0.0005580082652159035
DEBUG:root:i=136 residual=0.0005489647155627608
DEBUG:root:i=137 residual=0.0005401236121542752
DEBUG:root:i=138 residual=0.0005314827430993319
DEBUG:root:i=139 residual=0.0005230280221439898
DEBUG:root:i=140 residual=0.0005147632327862084
DEBUG:root:i=141 residual=0.0005066782468929887
DEBUG:root:i=142 residual=0.0004987612483091652
DEBUG:root:i=143 residual=0.0004910099669359624
DEBUG:root:i=144 residual=0.00048342355876229703
DEBUG:root:i=145 residual=0.0004759930889122188
DEBUG:root:i=146 residual=0.00046871404629200697
DEBUG:root:i=147 residual=0.00046158937038853765
DEBUG:root:i=148 residual=0.00045460049295797944
DEBUG:root:i=149 residual=0.0004477553302422166
DEBUG:root:i=150 residual=0.00044104462722316384
DEBUG:root:i=151 residual=0.0004344619228504598
DEBUG:root:i=152 residual=0.0004280243010725826
DEBUG:root:i=153 residual=0.0004216895904392004
DEBUG:root:i=154 residual=0.0004154883208684623
DEBUG:root:i=155 residual=0.0004094111209269613
DEBUG:root:i=156 residual=0.00040343767614103854
DEBUG:root:i=157 residual=0.0003975781728513539
DEBUG:root:i=158 residual=0.0003918301372323185
DEBUG:root:i=159 residual=0.0003861833247356117
DEBUG:root:i=160 residual=0.0003806366876233369
DEBUG:root:i=161 residual=0.00037520111072808504
DEBUG:root:i=162 residual=0.0003698597720358521
DEBUG:root:i=163 residual=0.0003646108671091497
DEBUG:root:i=164 residual=0.00035945873241871595
DEBUG:root:i=165 residual=0.0003543980128597468
DEBUG:root:i=166 residual=0.0003494229749776423
DEBUG:root:i=167 residual=0.00034453943953849375
DEBUG:root:i=168 residual=0.000339737453032285
DEBUG:root:i=169 residual=0.0003350218466948718
DEBUG:root:i=170 residual=0.00033038321998901665
DEBUG:root:i=171 residual=0.00032583027496002614
DEBUG:root:i=172 residual=0.00032134202774614096
DEBUG:root:i=173 residual=0.00031694050994701684
DEBUG:root:i=174 residual=0.0003126032534055412
DEBUG:root:i=175 residual=0.0003083454503212124
DEBUG:root:i=176 residual=0.00030415726359933615
DEBUG:root:i=177 residual=0.00030003191204741597
DEBUG:root:i=178 residual=0.00029598359833471477
DEBUG:root:i=179 residual=0.0002919979742728174
DEBUG:root:i=180 residual=0.0002880769898183644
DEBUG:root:i=181 residual=0.00028421488241292536
DEBUG:root:i=182 residual=0.00028041479527018964
DEBUG:root:i=183 residual=0.0002766780380625278
DEBUG:root:i=184 residual=0.00027299567591398954
DEBUG:root:i=185 residual=0.00026937961229123175
DEBUG:root:i=186 residual=0.0002658182056620717
DEBUG:root:i=187 residual=0.00026230834191665053
DEBUG:root:i=188 residual=0.0002588558418210596
DEBUG:root:i=189 residual=0.0002554564562160522
DEBUG:root:i=190 residual=0.0002521056740079075
DEBUG:root:i=191 residual=0.0002488085883669555
DEBUG:root:i=192 residual=0.0002455666835885495
DEBUG:root:i=193 residual=0.00024237073375843465
DEBUG:root:i=194 residual=0.00023921980755403638
DEBUG:root:i=195 residual=0.000236122592468746
DEBUG:root:i=196 residual=0.00023306840739678591
DEBUG:root:i=197 residual=0.00023005953698884696
DEBUG:root:i=198 residual=0.00022709849872626364
DEBUG:root:i=199 residual=0.00022417826403398067
DEBUG:root:i=200 residual=0.00022130364959593862
DEBUG:root:i=201 residual=0.0002184729091823101
DEBUG:root:i=202 residual=0.00021567920339293778
DEBUG:root:i=203 residual=0.00021292864403221756
DEBUG:root:i=204 residual=0.00021021901920903474
DEBUG:root:i=205 residual=0.00020754650176968426
DEBUG:root:i=206 residual=0.00020491291070356965
DEBUG:root:i=207 residual=0.00020231878443155438
DEBUG:root:i=208 residual=0.0001997616491280496
DEBUG:root:i=209 residual=0.0001972411118913442
DEBUG:root:i=210 residual=0.00019475712906569242
DEBUG:root:i=211 residual=0.0001923059462569654
DEBUG:root:i=212 residual=0.0001898919144878164
DEBUG:root:i=213 residual=0.00018751305469777435
DEBUG:root:i=214 residual=0.00018516524869482964
DEBUG:root:i=215 residual=0.00018284947145730257
DEBUG:root:i=216 residual=0.00018057337729260325
DEBUG:root:i=217 residual=0.00017832094454206526
DEBUG:root:i=218 residual=0.00017610321810934693
DEBUG:root:i=219 residual=0.00017391482833772898
DEBUG:root:i=220 residual=0.00017175704124383628
DEBUG:root:i=221 residual=0.00016963040980044752
DEBUG:root:i=222 residual=0.00016753468662500381
DEBUG:root:i=223 residual=0.000165466612088494
DEBUG:root:i=224 residual=0.000163423886988312
DEBUG:root:i=225 residual=0.00016141073137987405
DEBUG:root:i=226 residual=0.00015942662139423192
DEBUG:root:i=227 residual=0.00015747170255053788
DEBUG:root:i=228 residual=0.00015553818957414478
DEBUG:root:i=229 residual=0.00015363161219283938
DEBUG:root:i=230 residual=0.00015175386215560138
DEBUG:root:i=231 residual=0.00014990098134148866
DEBUG:root:i=232 residual=0.00014807020488660783
DEBUG:root:i=233 residual=0.0001462665240978822
DEBUG:root:i=234 residual=0.00014448593719862401
DEBUG:root:i=235 residual=0.0001427272945875302
DEBUG:root:i=236 residual=0.00014099632971920073
DEBUG:root:i=237 residual=0.00013928486441727728
DEBUG:root:i=238 residual=0.00013759774446953088
DEBUG:root:i=239 residual=0.00013593259791377932
DEBUG:root:i=240 residual=0.0001342905015917495
DEBUG:root:i=241 residual=0.00013266844325698912
DEBUG:root:i=242 residual=0.0001310696970904246
DEBUG:root:i=243 residual=0.00012948863150086254
DEBUG:root:i=244 residual=0.00012792926281690598
DEBUG:root:i=245 residual=0.00012639253691304475
DEBUG:root:i=246 residual=0.0001248744665645063
DEBUG:root:i=247 residual=0.00012337451335042715
DEBUG:root:i=248 residual=0.00012189672270324081
DEBUG:root:i=249 residual=0.00012043571041431278
DEBUG:root:i=250 residual=0.00011899833043571562
DEBUG:root:i=251 residual=0.00011757469474105164
DEBUG:root:i=252 residual=0.00011617183190537617
DEBUG:root:i=253 residual=0.0001147840594057925
DEBUG:root:i=254 residual=0.0001134183257818222
DEBUG:root:i=255 residual=0.00011206524504814297
DEBUG:root:i=256 residual=0.00011073365749325603
DEBUG:root:i=257 residual=0.00010941584332613274
DEBUG:root:i=258 residual=0.00010811690299306065
DEBUG:root:i=259 residual=0.00010683362052077428
DEBUG:root:i=260 residual=0.00010556954657658935
DEBUG:root:i=261 residual=0.00010431528062326834
DEBUG:root:i=262 residual=0.00010308304626960307
DEBUG:root:i=263 residual=0.00010186240979237482
DEBUG:root:i=264 residual=0.00010065696551464498
DEBUG:root:i=265 residual=9.946725913323462e-05
DEBUG:root:i=266 residual=9.829495684243739e-05
DEBUG:root:i=267 residual=9.713746112538502e-05
DEBUG:root:i=268 residual=9.599185432307422e-05
DEBUG:root:i=269 residual=9.486301860306412e-05
DEBUG:root:i=270 residual=9.374600631417707e-05
DEBUG:root:i=271 residual=9.264428081223741e-05
DEBUG:root:i=272 residual=9.155869338428602e-05
DEBUG:root:i=273 residual=9.048108768183738e-05
DEBUG:root:i=274 residual=8.94204931682907e-05
DEBUG:root:i=275 residual=8.837351197144017e-05
DEBUG:root:i=276 residual=8.733998402021825e-05
DEBUG:root:i=277 residual=8.631739183329046e-05
DEBUG:root:i=278 residual=8.530886407243088e-05
DEBUG:root:i=279 residual=8.4309620433487e-05
DEBUG:root:i=280 residual=8.332610741490498e-05
DEBUG:root:i=281 residual=8.235326095018536e-05
DEBUG:root:i=282 residual=8.139387500705197e-05
DEBUG:root:i=283 residual=8.044460992095992e-05
DEBUG:root:i=284 residual=7.950719009386376e-05
DEBUG:root:i=285 residual=7.858227036194876e-05
DEBUG:root:i=286 residual=7.7668039011769e-05
DEBUG:root:i=287 residual=7.676617678953335e-05
DEBUG:root:i=288 residual=7.587356230942532e-05
DEBUG:root:i=289 residual=7.49933606130071e-05
DEBUG:root:i=290 residual=7.412352715618908e-05
DEBUG:root:i=291 residual=7.326342165470123e-05
DEBUG:root:i=292 residual=7.241291314130649e-05
DEBUG:root:i=293 residual=7.157443178584799e-05
DEBUG:root:i=294 residual=7.074546010699123e-05
DEBUG:root:i=295 residual=6.99267620802857e-05
DEBUG:root:i=296 residual=6.911720265634358e-05
DEBUG:root:i=297 residual=6.831931386841461e-05
DEBUG:root:i=298 residual=6.753092748112977e-05
DEBUG:root:i=299 residual=6.675203621853143e-05
DEBUG:root:i=300 residual=6.598173786187544e-05
DEBUG:root:i=301 residual=6.521954492200166e-05
DEBUG:root:i=302 residual=6.446746556321159e-05
DEBUG:root:i=303 residual=6.372470670612529e-05
DEBUG:root:i=304 residual=6.299350206973031e-05
DEBUG:root:i=305 residual=6.226700497791171e-05
DEBUG:root:i=306 residual=6.155100709293038e-05
DEBUG:root:i=307 residual=6.08419140917249e-05
DEBUG:root:i=308 residual=6.014273458276875e-05
DEBUG:root:i=309 residual=5.945220618741587e-05
DEBUG:root:i=310 residual=5.876793875358999e-05
DEBUG:root:i=311 residual=5.809389767819084e-05
DEBUG:root:i=312 residual=5.742593202739954e-05
DEBUG:root:i=313 residual=5.676883301930502e-05
DEBUG:root:i=314 residual=5.611762753687799e-05
DEBUG:root:i=315 residual=5.547431283048354e-05
DEBUG:root:i=316 residual=5.484063149197027e-05
DEBUG:root:i=317 residual=5.420882735052146e-05
DEBUG:root:i=318 residual=5.359070564736612e-05
DEBUG:root:i=319 residual=5.2976909501012415e-05
DEBUG:root:i=320 residual=5.237200821284205e-05
DEBUG:root:i=321 residual=5.177432103664614e-05
DEBUG:root:i=322 residual=5.118073386256583e-05
DEBUG:root:i=323 residual=5.059866452938877e-05
DEBUG:root:i=324 residual=5.00207461300306e-05
DEBUG:root:i=325 residual=4.9450456572230905e-05
DEBUG:root:i=326 residual=4.888595503871329e-05
DEBUG:root:i=327 residual=4.83286967210006e-05
DEBUG:root:i=328 residual=4.777844515047036e-05
DEBUG:root:i=329 residual=4.723293022834696e-05
DEBUG:root:i=330 residual=4.669704139814712e-05
DEBUG:root:i=331 residual=4.616539808921516e-05
DEBUG:root:i=332 residual=4.563765833154321e-05
DEBUG:root:i=333 residual=4.511920269578695e-05
DEBUG:root:i=334 residual=4.460684067453258e-05
DEBUG:root:i=335 residual=4.409705434227362e-05
DEBUG:root:i=336 residual=4.35980582551565e-05
DEBUG:root:i=337 residual=4.310241274652071e-05
DEBUG:root:i=338 residual=4.2611471144482493e-05
DEBUG:root:i=339 residual=4.212802741676569e-05
DEBUG:root:i=340 residual=4.165048085269518e-05
DEBUG:root:i=341 residual=4.1177285311277956e-05
DEBUG:root:i=342 residual=4.070904469699599e-05
DEBUG:root:i=343 residual=4.024925146950409e-05
DEBUG:root:i=344 residual=3.9792350435163826e-05
DEBUG:root:i=345 residual=3.934006599592976e-05
DEBUG:root:i=346 residual=3.8896192563697696e-05
DEBUG:root:i=347 residual=3.845602259389125e-05
DEBUG:root:i=348 residual=3.802099308813922e-05
DEBUG:root:i=349 residual=3.758764796657488e-05
DEBUG:root:i=350 residual=3.7164794775890186e-05
DEBUG:root:i=351 residual=3.674569234135561e-05
DEBUG:root:i=352 residual=3.632740845205262e-05
DEBUG:root:i=353 residual=3.591868880903348e-05
DEBUG:root:i=354 residual=3.551082409103401e-05
DEBUG:root:i=355 residual=3.511148679535836e-05
DEBUG:root:i=356 residual=3.4714896173682064e-05
DEBUG:root:i=357 residual=3.432335870456882e-05
DEBUG:root:i=358 residual=3.393441511434503e-05
DEBUG:root:i=359 residual=3.354987347847782e-05
DEBUG:root:i=360 residual=3.317254959256388e-05
DEBUG:root:i=361 residual=3.2798427128000185e-05
DEBUG:root:i=362 residual=3.242950333515182e-05
DEBUG:root:i=363 residual=3.206377732567489e-05
DEBUG:root:i=364 residual=3.170215859427117e-05
DEBUG:root:i=365 residual=3.134557118755765e-05
DEBUG:root:i=366 residual=3.098875458817929e-05
DEBUG:root:i=367 residual=3.064165866817348e-05
DEBUG:root:i=368 residual=3.02964272123063e-05
DEBUG:root:i=369 residual=2.9955470381537452e-05
DEBUG:root:i=370 residual=2.9619000997627154e-05
DEBUG:root:i=371 residual=2.928550929937046e-05
DEBUG:root:i=372 residual=2.8955222660442814e-05
DEBUG:root:i=373 residual=2.8629927328438498e-05
DEBUG:root:i=374 residual=2.8309408662607893e-05
DEBUG:root:i=375 residual=2.7991074603050947e-05
DEBUG:root:i=376 residual=2.7676636818796396e-05
DEBUG:root:i=377 residual=2.736540227488149e-05
DEBUG:root:i=378 residual=2.7053747544414364e-05
DEBUG:root:i=379 residual=2.6750292818178423e-05
DEBUG:root:i=380 residual=2.6452256861375645e-05
DEBUG:root:i=381 residual=2.6152762075071223e-05
DEBUG:root:i=382 residual=2.5859590095933527e-05
DEBUG:root:i=383 residual=2.5567944248905405e-05
DEBUG:root:i=384 residual=2.528349432395771e-05
DEBUG:root:i=385 residual=2.4998706066980958e-05
DEBUG:root:i=386 residual=2.471877814969048e-05
DEBUG:root:i=387 residual=2.444048732286319e-05
DEBUG:root:i=388 residual=2.416765710222535e-05
DEBUG:root:i=389 residual=2.3895463527878746e-05
DEBUG:root:i=390 residual=2.3628923372598365e-05
DEBUG:root:i=391 residual=2.3364496883004904e-05
DEBUG:root:i=392 residual=2.310197305632755e-05
DEBUG:root:i=393 residual=2.284377478645183e-05
DEBUG:root:i=394 residual=2.25863914238289e-05
DEBUG:root:i=395 residual=2.2333633751259185e-05
DEBUG:root:i=396 residual=2.2083811927586794e-05
DEBUG:root:i=397 residual=2.1836935047758743e-05
DEBUG:root:i=398 residual=2.159174437110778e-05
DEBUG:root:i=399 residual=2.1351705072447658e-05
DEBUG:root:i=400 residual=2.1111403839313425e-05
DEBUG:root:i=401 residual=2.0875631889794022e-05
DEBUG:root:i=402 residual=2.064183536276687e-05
DEBUG:root:i=403 residual=2.041173866018653e-05
DEBUG:root:i=404 residual=2.0182609659968875e-05
DEBUG:root:i=405 residual=1.9956343749072403e-05
DEBUG:root:i=406 residual=1.973528924281709e-05
DEBUG:root:i=407 residual=1.951332887983881e-05
DEBUG:root:i=408 residual=1.9296212485642172e-05
DEBUG:root:i=409 residual=1.9079243429587223e-05
DEBUG:root:i=410 residual=1.8865401216316968e-05
DEBUG:root:i=411 residual=1.8657367036212236e-05
DEBUG:root:i=412 residual=1.844652979343664e-05
DEBUG:root:i=413 residual=1.824129867600277e-05
DEBUG:root:i=414 residual=1.803816485335119e-05
DEBUG:root:i=415 residual=1.78370773937786e-05
DEBUG:root:i=416 residual=1.7636335542192683e-05
DEBUG:root:i=417 residual=1.7441696400055662e-05
DEBUG:root:i=418 residual=1.7244678019778803e-05
DEBUG:root:i=419 residual=1.7054397176252678e-05
DEBUG:root:i=420 residual=1.6862859411048703e-05
DEBUG:root:i=421 residual=1.667420110607054e-05
DEBUG:root:i=422 residual=1.6488393157487735e-05
DEBUG:root:i=423 residual=1.6305057215504348e-05
DEBUG:root:i=424 residual=1.6124220564961433e-05
DEBUG:root:i=425 residual=1.5942678146529943e-05
DEBUG:root:i=426 residual=1.5764846466481686e-05
DEBUG:root:i=427 residual=1.559002157591749e-05
DEBUG:root:i=428 residual=1.541605706734117e-05
DEBUG:root:i=429 residual=1.5243749658111483e-05
DEBUG:root:i=430 residual=1.5073834219947457e-05
DEBUG:root:i=431 residual=1.490747308707796e-05
DEBUG:root:i=432 residual=1.4740101505594794e-05
DEBUG:root:i=433 residual=1.4577824003936257e-05
DEBUG:root:i=434 residual=1.4414136785489973e-05
DEBUG:root:i=435 residual=1.4254726920626126e-05
DEBUG:root:i=436 residual=1.4094317521085031e-05
DEBUG:root:i=437 residual=1.3937678886577487e-05
DEBUG:root:i=438 residual=1.3782791029370856e-05
DEBUG:root:i=439 residual=1.3627484804601409e-05
DEBUG:root:i=440 residual=1.347703619103413e-05
DEBUG:root:i=441 residual=1.3327518900041468e-05
DEBUG:root:i=442 residual=1.3179738743929192e-05
DEBUG:root:i=443 residual=1.3033363757131156e-05
DEBUG:root:i=444 residual=1.2888700439361855e-05
DEBUG:root:i=445 residual=1.2744842933898326e-05
DEBUG:root:i=446 residual=1.2601714843185619e-05
DEBUG:root:i=447 residual=1.246362353413133e-05
DEBUG:root:i=448 residual=1.232470094691962e-05
DEBUG:root:i=449 residual=1.2187620995973703e-05
DEBUG:root:i=450 residual=1.2051753401465248e-05
DEBUG:root:i=451 residual=1.1917573829123285e-05
DEBUG:root:i=452 residual=1.178790262201801e-05
DEBUG:root:i=453 residual=1.1652959074126557e-05
DEBUG:root:i=454 residual=1.1523989087436348e-05
DEBUG:root:i=455 residual=1.1396488844184205e-05
DEBUG:root:i=456 residual=1.12708039523568e-05
DEBUG:root:i=457 residual=1.1146055840072222e-05
DEBUG:root:i=458 residual=1.102185979107162e-05
DEBUG:root:i=459 residual=1.0898659638769459e-05
DEBUG:root:i=460 residual=1.0777239367598668e-05
DEBUG:root:i=461 residual=1.0656413905962836e-05
DEBUG:root:i=462 residual=1.0540124094404746e-05
DEBUG:root:i=463 residual=1.042332860379247e-05
DEBUG:root:i=464 residual=1.0306505828339141e-05
DEBUG:root:i=465 residual=1.0193030902883038e-05
DEBUG:root:i=466 residual=1.0080171705340035e-05
DEBUG:root:i=467 residual=9.966904144675937e-06
DEBUG:root:i=468 residual=9.8574137155083e-06
DEBUG:root:i=469 residual=9.747694093675818e-06
DEBUG:root:i=470 residual=9.63922866503708e-06
DEBUG:root:i=471 residual=9.532219337415881e-06
DEBUG:root:i=472 residual=9.426054020877928e-06
DEBUG:root:i=473 residual=9.322921869170386e-06
DEBUG:root:i=474 residual=9.218200830218848e-06
DEBUG:root:i=475 residual=9.115557077166159e-06
DEBUG:root:i=476 residual=9.014789611683227e-06
DEBUG:root:i=477 residual=8.913969395507593e-06
DEBUG:root:i=478 residual=8.815197361400351e-06
DEBUG:root:i=479 residual=8.717651326151099e-06
DEBUG:root:i=480 residual=8.621909728390165e-06
DEBUG:root:i=481 residual=8.52578796184389e-06
DEBUG:root:i=482 residual=8.431833521171939e-06
DEBUG:root:i=483 residual=8.339134183188435e-06
DEBUG:root:i=484 residual=8.24496237328276e-06
DEBUG:root:i=485 residual=8.153729140758514e-06
DEBUG:root:i=486 residual=8.063648238021415e-06
DEBUG:root:i=487 residual=7.973470928845927e-06
DEBUG:root:i=488 residual=7.886511411925312e-06
DEBUG:root:i=489 residual=7.806170287949499e-06
DEBUG:root:i=490 residual=7.712634214840364e-06
DEBUG:root:i=491 residual=7.625677426403854e-06
DEBUG:root:i=492 residual=7.5418920459924266e-06
DEBUG:root:i=493 residual=7.458510935975937e-06
DEBUG:root:i=494 residual=7.376509074674686e-06
DEBUG:root:i=495 residual=7.2958537202794105e-06
DEBUG:root:i=496 residual=7.214499419205822e-06
DEBUG:root:i=497 residual=7.132638984330697e-06
DEBUG:root:i=498 residual=7.054062734823674e-06
DEBUG:root:i=499 residual=6.974643838475458e-06
DEBUG:root:i=500 residual=6.898520496179117e-06
DEBUG:root:i=501 residual=6.8216036197554786e-06
DEBUG:root:i=502 residual=6.748204668838298e-06
DEBUG:root:i=503 residual=6.669833965133876e-06
DEBUG:root:i=504 residual=6.597741048608441e-06
DEBUG:root:i=505 residual=6.525483968289336e-06
DEBUG:root:i=506 residual=6.451829449360957e-06
DEBUG:root:i=507 residual=6.3809425228100736e-06
DEBUG:root:i=508 residual=6.309979198704241e-06
DEBUG:root:i=509 residual=6.239509730221471e-06
DEBUG:root:i=510 residual=6.1712844399153255e-06
DEBUG:root:i=511 residual=6.103485247876961e-06
DEBUG:root:i=512 residual=6.035053047526162e-06
DEBUG:root:i=513 residual=5.967488505120855e-06
DEBUG:root:i=514 residual=5.904350928176427e-06
DEBUG:root:i=515 residual=5.8364080359751824e-06
DEBUG:root:i=516 residual=5.773174052592367e-06
DEBUG:root:i=517 residual=5.70582506043138e-06
DEBUG:root:i=518 residual=5.644523298542481e-06
DEBUG:root:i=519 residual=5.583474376180675e-06
DEBUG:root:i=520 residual=5.519512797036441e-06
DEBUG:root:i=521 residual=5.4604083743470255e-06
DEBUG:root:i=522 residual=5.398471330408938e-06
DEBUG:root:i=523 residual=5.339244125934783e-06
DEBUG:root:i=524 residual=5.279358447296545e-06
DEBUG:root:i=525 residual=5.2225127546989825e-06
DEBUG:root:i=526 residual=5.162101388123119e-06
DEBUG:root:i=527 residual=5.105696345708566e-06
DEBUG:root:i=528 residual=5.049356332165189e-06
DEBUG:root:i=529 residual=4.995694325771183e-06
DEBUG:root:i=530 residual=4.939343398291385e-06
DEBUG:root:i=531 residual=4.883762358076638e-06
DEBUG:root:i=532 residual=4.830590569326887e-06
DEBUG:root:i=533 residual=4.776921741722617e-06
DEBUG:root:i=534 residual=4.725246981251985e-06
DEBUG:root:i=535 residual=4.6716208998986986e-06
DEBUG:root:i=536 residual=4.62173329651705e-06
DEBUG:root:i=537 residual=4.569188604364172e-06
DEBUG:root:i=538 residual=4.518284185905941e-06
DEBUG:root:i=539 residual=4.470245130505646e-06
DEBUG:root:i=540 residual=4.418746357259806e-06
DEBUG:root:i=541 residual=4.369112048152601e-06
DEBUG:root:i=542 residual=4.329201146902051e-06
DEBUG:root:i=543 residual=4.273243121133419e-06
DEBUG:root:i=544 residual=4.227403678669361e-06
DEBUG:root:i=545 residual=4.1824032450676896e-06
DEBUG:root:i=546 residual=4.136352799832821e-06
DEBUG:root:i=547 residual=4.088138666702434e-06
DEBUG:root:i=548 residual=4.042563432449242e-06
DEBUG:root:i=549 residual=3.999048658442916e-06
DEBUG:root:i=550 residual=3.953869963879697e-06
DEBUG:root:i=551 residual=3.912566626240732e-06
DEBUG:root:i=552 residual=3.866643055516761e-06
DEBUG:root:i=553 residual=3.832073161902372e-06
DEBUG:root:i=554 residual=3.783704414672684e-06
DEBUG:root:i=555 residual=3.7400739074655576e-06
DEBUG:root:i=556 residual=3.7003751458541956e-06
DEBUG:root:i=557 residual=3.6568678751791595e-06
DEBUG:root:i=558 residual=3.6195540360495215e-06
DEBUG:root:i=559 residual=3.5774442039837595e-06
DEBUG:root:i=560 residual=3.538800456226454e-06
DEBUG:root:i=561 residual=3.4984652756975265e-06
DEBUG:root:i=562 residual=3.461664618953364e-06
DEBUG:root:i=563 residual=3.421909468670492e-06
DEBUG:root:i=564 residual=3.3845276448118966e-06
DEBUG:root:i=565 residual=3.3475660075055202e-06
DEBUG:root:i=566 residual=3.3101921417255653e-06
DEBUG:root:i=567 residual=3.272329195169732e-06
DEBUG:root:i=568 residual=3.239359330109437e-06
DEBUG:root:i=569 residual=3.202432708349079e-06
DEBUG:root:i=570 residual=3.1656752526032506e-06
DEBUG:root:i=571 residual=3.1317661068896996e-06
DEBUG:root:i=572 residual=3.0965657060733065e-06
DEBUG:root:i=573 residual=3.06382344206213e-06
DEBUG:root:i=574 residual=3.0288931611721637e-06
DEBUG:root:i=575 residual=2.9959564926684834e-06
DEBUG:root:i=576 residual=2.961595100714476e-06
DEBUG:root:i=577 residual=2.9307350359886186e-06
DEBUG:root:i=578 residual=2.8982231015106663e-06
DEBUG:root:i=579 residual=2.8651072625507368e-06
DEBUG:root:i=580 residual=2.833482994901715e-06
DEBUG:root:i=581 residual=2.801818709485815e-06
DEBUG:root:i=582 residual=2.7697551558958367e-06
DEBUG:root:i=583 residual=2.740861191341537e-06
DEBUG:root:i=584 residual=2.7094724828202743e-06
DEBUG:root:i=585 residual=2.6806121695699403e-06
DEBUG:root:i=586 residual=2.6503812478040345e-06
DEBUG:root:i=587 residual=2.6202251319773495e-06
DEBUG:root:i=588 residual=2.594755414975225e-06
DEBUG:root:i=589 residual=2.5624378849897766e-06
DEBUG:root:i=590 residual=2.5357223876198987e-06
DEBUG:root:i=591 residual=2.5079700662899995e-06
DEBUG:root:i=592 residual=2.4838943772920175e-06
DEBUG:root:i=593 residual=2.450681677146349e-06
DEBUG:root:i=594 residual=2.4240525817731395e-06
DEBUG:root:i=595 residual=2.400322955509182e-06
DEBUG:root:i=596 residual=2.374058794885059e-06
DEBUG:root:i=597 residual=2.3484640223614406e-06
DEBUG:root:i=598 residual=2.3205845991469687e-06
DEBUG:root:i=599 residual=2.2940751023270423e-06
DEBUG:root:i=600 residual=2.268972821184434e-06
DEBUG:root:i=601 residual=2.2455458292824915e-06
DEBUG:root:i=602 residual=2.2212875592231285e-06
DEBUG:root:i=603 residual=2.1943528736301232e-06
DEBUG:root:i=604 residual=2.169090521420003e-06
DEBUG:root:i=605 residual=2.1462813037942396e-06
DEBUG:root:i=606 residual=2.1218625079200137e-06
DEBUG:root:i=607 residual=2.0996858438593335e-06
DEBUG:root:i=608 residual=2.0770808077941183e-06
DEBUG:root:i=609 residual=2.0532156668195967e-06
DEBUG:root:i=610 residual=2.0317208964115707e-06
DEBUG:root:i=611 residual=2.0090753878321266e-06
DEBUG:root:i=612 residual=1.985963308470673e-06
DEBUG:root:i=613 residual=1.963053819054039e-06
DEBUG:root:i=614 residual=1.944788209584658e-06
DEBUG:root:i=615 residual=1.9217466160625918e-06
DEBUG:root:i=616 residual=1.9015792531718034e-06
DEBUG:root:i=617 residual=1.8772659586829832e-06
DEBUG:root:i=618 residual=1.8602257796374033e-06
DEBUG:root:i=619 residual=1.8382984308118466e-06
DEBUG:root:i=620 residual=1.8161765638069483e-06
DEBUG:root:i=621 residual=1.7998371504290844e-06
DEBUG:root:i=622 residual=1.7807037693273742e-06
DEBUG:root:i=623 residual=1.7591002006156486e-06
DEBUG:root:i=624 residual=1.7390415223417222e-06
DEBUG:root:i=625 residual=1.7239958651771303e-06
DEBUG:root:i=626 residual=1.7016744777720305e-06
DEBUG:root:i=627 residual=1.681502681094571e-06
DEBUG:root:i=628 residual=1.6633347286187927e-06
DEBUG:root:i=629 residual=1.6445288792965584e-06
DEBUG:root:i=630 residual=1.6271329741357476e-06
DEBUG:root:i=631 residual=1.6093376871140208e-06
DEBUG:root:i=632 residual=1.5914375808279146e-06
DEBUG:root:i=633 residual=1.5751656974316575e-06
DEBUG:root:i=634 residual=1.5578214060951723e-06
DEBUG:root:i=635 residual=1.538241235721216e-06
DEBUG:root:i=636 residual=1.5212910966511117e-06
DEBUG:root:i=637 residual=1.507011120338575e-06
DEBUG:root:i=638 residual=1.493864488111285e-06
DEBUG:root:i=639 residual=1.4738235449840431e-06
DEBUG:root:i=640 residual=1.461107444811205e-06
DEBUG:root:i=641 residual=1.4398823395822546e-06
DEBUG:root:i=642 residual=1.421787260369456e-06
DEBUG:root:i=643 residual=1.4076240404392593e-06
DEBUG:root:i=644 residual=1.391810769746371e-06
DEBUG:root:i=645 residual=1.3794631286145886e-06
DEBUG:root:i=646 residual=1.362667035209597e-06
DEBUG:root:i=647 residual=1.3519969570552348e-06
DEBUG:root:i=648 residual=1.332184638158651e-06
DEBUG:root:i=649 residual=1.319901343777019e-06
DEBUG:root:i=650 residual=1.3028957255301066e-06
DEBUG:root:i=651 residual=1.295206175200292e-06
DEBUG:root:i=652 residual=1.2761611287714913e-06
DEBUG:root:i=653 residual=1.2610701105586486e-06
DEBUG:root:i=654 residual=1.25090332403488e-06
DEBUG:root:i=655 residual=1.2345365121291252e-06
DEBUG:root:i=656 residual=1.2193041811769945e-06
DEBUG:root:i=657 residual=1.206781803375634e-06
DEBUG:root:i=658 residual=1.1928178764719632e-06
DEBUG:root:i=659 residual=1.1792570830948534e-06
DEBUG:root:i=660 residual=1.1674580946419155e-06
DEBUG:root:i=661 residual=1.1526738035172457e-06
DEBUG:root:i=662 residual=1.1447965562183526e-06
DEBUG:root:i=663 residual=1.130549208028242e-06
DEBUG:root:i=664 residual=1.1230723657718045e-06
DEBUG:root:i=665 residual=1.1111375215477892e-06
DEBUG:root:i=666 residual=1.0923156423814362e-06
DEBUG:root:i=667 residual=1.0807121952893795e-06
DEBUG:root:i=668 residual=1.067076482286211e-06
DEBUG:root:i=669 residual=1.055256348081457e-06
DEBUG:root:i=670 residual=1.0428547057017568e-06
DEBUG:root:i=671 residual=1.0342727136958274e-06
DEBUG:root:i=672 residual=1.0207188552158186e-06
DEBUG:root:i=673 residual=1.0109986305906205e-06
DEBUG:root:i=674 residual=9.998100267694099e-07
INFO:root:rank=0 pagerank=7.0147e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 pagerank=7.0147e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 pagerank=1.0537e-01 url=www.lawfareblog.com/cost-using-zero-days
INFO:root:rank=3 pagerank=3.1757e-02 url=www.lawfareblog.com/lawfare-podcast-former-congressman-brian-baird-and-daniel-schuman-how-congress-can-continue-function
INFO:root:rank=4 pagerank=2.2269e-02 url=www.lawfareblog.com/events
INFO:root:rank=5 pagerank=1.6050e-02 url=www.lawfareblog.com/water-wars-increased-us-focus-indo-pacific
INFO:root:rank=6 pagerank=1.6049e-02 url=www.lawfareblog.com/water-wars-drill-maybe-drill
INFO:root:rank=7 pagerank=1.6045e-02 url=www.lawfareblog.com/water-wars-disjointed-operations-south-china-sea
INFO:root:rank=8 pagerank=1.6042e-02 url=www.lawfareblog.com/water-wars-song-oil-and-fire
INFO:root:rank=9 pagerank=1.6042e-02 url=www.lawfareblog.com/water-wars-sinking-feeling-philippine-china-relations
   ```

   Task 2, part 1:
   ```
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --personalization_vector_query='corona'
   INFO:root:rank=0 pagerank=6.3127e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 pagerank=6.3124e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 pagerank=1.5947e-01 url=www.lawfareblog.com/chinatalk-how-party-takes-its-propaganda-global
INFO:root:rank=3 pagerank=1.2209e-01 url=www.lawfareblog.com/brexit-not-immune-coronavirus
INFO:root:rank=4 pagerank=1.2209e-01 url=www.lawfareblog.com/rational-security-my-corona-edition
INFO:root:rank=5 pagerank=9.3360e-02 url=www.lawfareblog.com/trump-cant-reopen-country-over-state-objections
INFO:root:rank=6 pagerank=9.1920e-02 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
INFO:root:rank=7 pagerank=9.1920e-02 url=www.lawfareblog.com/britains-coronavirus-response
INFO:root:rank=8 pagerank=7.7770e-02 url=www.lawfareblog.com/lawfare-podcast-united-nations-and-coronavirus-crisis
INFO:root:rank=9 pagerank=7.2888e-02 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
   ```

   Task 2, part 2:
   ```
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --personalization_vector_query='corona' --search_query='-corona'
   INFO:root:rank=0 pagerank=6.3127e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 pagerank=6.3124e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 pagerank=1.5947e-01 url=www.lawfareblog.com/chinatalk-how-party-takes-its-propaganda-global
INFO:root:rank=3 pagerank=9.3360e-02 url=www.lawfareblog.com/trump-cant-reopen-country-over-state-objections
INFO:root:rank=4 pagerank=7.0277e-02 url=www.lawfareblog.com/fault-lines-foreign-policy-quarantined
INFO:root:rank=5 pagerank=6.9713e-02 url=www.lawfareblog.com/lawfare-podcast-mom-and-dad-talk-clinical-trials-pandemic
INFO:root:rank=6 pagerank=6.4944e-02 url=www.lawfareblog.com/limits-world-health-organization
INFO:root:rank=7 pagerank=5.9492e-02 url=www.lawfareblog.com/chinatalk-dispatches-shanghai-beijing-and-hong-kong
INFO:root:rank=8 pagerank=5.1245e-02 url=www.lawfareblog.com/us-moves-dismiss-case-against-company-linked-ira-troll-farm
INFO:root:rank=9 pagerank=5.1245e-02 url=www.lawfareblog.com/livestream-house-armed-services-holds-hearing-national-security-challenges-north-and-south-america
   ```

1. Ensure that all your changes to the `pagerank.py` and `README.md` files are committed to your repo and pushed to github.

1. Get at least 5 stars on your repo.
   (You may trade stars with other students in the class.)

   > **NOTE:**
   > 
   > Recruiters use github profiles to determine who to hire,
   > and pagerank is used to rank user profiles and projects.
   > Links in this graph correspond to who has starred/followed who's repo.
   > By getting more stars on your repo, you'll be increasing your github pagerank, which increases the likelihood that recruiters will hire you.
   > To see an example, [perform a search for `data mining`](https://github.com/search?q=data+mining).
   > Notice that the results are returned "approximately" ranked by the number of stars,
   > but because "some stars count more than others" the results are not exactly ranked by the number of stars.
   > (I asked you not to fork this repo because forks are ranked lower than non-forks.)
   >
   > In some sense, we are doing a "dual problem" to data mining by getting these stars.
   > Recruiters are using data mining to find out who the best people to recruit are,
   > and we are hacking their data mining algorithms by making those algorithms select you instead of someone else.
   >
   > If you're interested in exploring this idea further, here's a python tutorial for extracting GitHub's social graph: <https://www.oreilly.com/library/view/mining-the-social/9781449368180/ch07.html> ; if you're interested in learning more about how recruiters use github profiles, read this Hacker News post: <https://news.ycombinator.com/item?id=19413348>.

1. Submit the url of your repo to sakai.

   Each part is worth 2 points, for 12 points overall.
