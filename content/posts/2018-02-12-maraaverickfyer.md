---
title: How to maraaverickfy a blog post without even reading it
author: maelle
type: post
date: 2018-02-12T11:49:40+00:00
spacious_page_layout:
  - default_layout
categories:
  - Data Science
  - R
tags:
  - r

---

Steph is currently out of the office, teaching people cool Data Science stuff on a cruise at [Tech Outbound](http://www.techoutbound.com/view-events.html). She [counts on her team](https://twitter.com/LockeData/status/960831435245531141) to keep the company's Twitter account afloat in the meantime, so I had to think of a way to contribute. What about advertising existing content from her blog in the style of her Twitter role model [Mara Averick](https://twitter.com/dataandme), i.e. an informative tweet accompanied by appealing screenshots?

In this blog post I'll show how I can cover for Steph *without even reading her blog posts*, using R tools to summarize blog posts! Read further if you want to know more about `markovifyR`, `webshot` and `magick`, Steph's fantastic blog content, and my being lazy.

Summarizing a blog post in Steph's words
========================================

Can some NLP magic produce credible Stephspeak, and to be more precise Stephtechspeak? My strategy here is to 1) create a model for generating credible Stephspeak 2) use it to generate text based on the topic of the blog post. Steph is the best person to describe her own writing, and well, having tweets that sound like her will help her (data science) cruising go unnoticed.

I shall make use of the same method as [Katie Jolly in her blog post generating Rupi Kaur-style poems](http://katiejolly.io/blog/2018-01-05/random-rupi-markov-chain-poems) and [Julia Silge in her post about Stack Overflow super-contributer Jon Skeet](https://stackoverflow.blog/2018/01/15/thanks-million-jon-skeet/): fitting a Markov Chain model to existing and authentic text, and then using that model to generate new text.

In their posts, Katie and Julia used different packages, so [I wondered which one to choose](https://twitter.com/ma_salmon/status/956413367819915265). Alex Bresler's [`markovifyR`](https://github.com/abresler/markovifyR), a wrapper to the [Python library `markovify`](https://github.com/jsvine/markovify) won. Note that it depends on Python being installed in your system, which I didn't care about since my laptop already met that requirement since I started playing with the [`cleanNLP` package](https://github.com/statsmaths/cleanNLP).

Getting Steph's tweets
----------------------

I decided to use Steph's very own tweets as corpus, instead of also using Locke Data's tweets. I couldn't rely on `rtweet::get_timeline` because Twitter API only returns up to 3,200 tweets which is far less that Steph's whole production so I asked her to [request her Twitter archive](https://help.twitter.com/es/managing-your-account/how-to-download-your-twitter-archive) and to send it to me.

``` r
steph_tweets <- readr::read_csv("data/stefflocke_tweets.csv")
```

Fitting a Markov model
----------------------

The `markovifyR::generate_markovify_model` has several parameters to tweak the model, but I started by simply using the default ones, planning to change them if I wasn't happy with the output. Believe it or not, but it was my first time using `dplyr::pull`, I'm more used to using `$`. As a non-native speaker, I often am not able to correctly make sense of push/pull signs on doors but git and `dplyr` help me get used to the meaning of these words. I only kept original tweets, not the ones that were retweets from other accounts, and cleaned up the text using [this Stack Overflow thread](https://stackoverflow.com/questions/31348453/how-do-i-clean-twitter-data-in-r).

``` r
library("magrittr")
steph_text <- steph_tweets %>%
  dplyr::filter(is.na(retweeted_status_id)) %>%
  dplyr::pull(text) %>%
  stringr::str_replace_all("@[a-z,A-Z,0-9,\\_]*", "someone")  %>%
  stringr:: str_replace_all( "https:\\/\\/t.co/[a-z,A-Z,0-9]*","") %>%
  stringr:: str_replace_all( "http:\\/\\/t.co/[a-z,A-Z,0-9]*","") %>%
  trimws() 
markov_model <- markovifyR::generate_markovify_model(steph_text)
```

Generating new text based on the model
--------------------------------------

I wrote this function that creates new text *starting* with chosen words, returning `count` new sentences. Note that I set the random seed at the beginning of the function in order to make results reproducible. I'm using 200 instead of 240 as a limit on the number of characters because I wanted to be sure to have space for a (shortened) link to the blog post under scrutiny.

``` r
steph_speak <- function(start, markov_model, count, seed = 42){
  set.seed(seed)
  bot_tweets <- markovifyR::markovify_text(
  markov_model = markov_model,
  maximum_sentence_length = 200,
  start_words = start,
  output_column_name = 'stephbot_text',
  count = count,
  tries = 100,
  only_distinct = TRUE,
  return_message = TRUE)
bot_tweets$stephbot_text
}
```

Let's try out the function by generating text starting with "Check out" because it'd be a pretty natural start for a tweet promoting something.

``` r
steph_speak("Check out", markov_model, count = 3)
```

    ## [1] "Check out the concept."                   
    ## [2] "Check out Intro to R training environment"
    ## [3] "Check out this form!"

Another thing that Steph often says is "w00t" (in the 10090 tweets it comes up 55 times!).

``` r
steph_speak("W00t", markov_model, count =  3)
```

    ## [1] "W00t I have time for #satRdays already!"                                                   
    ## [2] "W00t someone talking #azurefunctions &amp; loved the framework, scalability, &amp; cost 2."
    ## [3] "W00t someone talking #AI"

Having replaced Twitter handles mentions with "someone" makes it look a bit weird. Nevertheless, I'm quite satisfied with the results which I find at least funny.

Getting data about Steph's blog posts
-------------------------------------

The actual goal here is to promote Steph's blog content. Over the year she has accumulated a nice collection of mostly technical blog posts. In code not shown here (but that I could share later), I used `rmarkdown::yaml_front_matter` to retrieve information from blog posts such as their titles and tags, and `xml2::read_xml` on the sitemap to retrieve all their URLs. When joining the two tables thus obtained, I got a nice table full of information about existing blog posts.

``` r
blog_info <- readr::read_csv("data/all_info_about_posts.csv")
blog_info <- tidyr::gather(blog_info, "tag", "value", 11:ncol(blog_info), na.rm = TRUE)
# remove the mark of category

blog_info <- dplyr::mutate(blog_info, tag = stringr::str_replace(tag, "cat\\_", ""))

# remove one tag
blog_info <- dplyr::filter(blog_info, ! tag %in% c("statuspost", "base"))
```

Creating text consistent with the post's topics
-----------------------------------------------

We now have a table of 800 rows with 177 unique tags/categories. For each post, I want to be able to generate a tweet text that sounds like Steph's speaking (thus using the aforementioned Markov model) and that looks at least slightly related to the blog post, thus using the tags. Now, the Markov model we've created will only help us generate new sentences starting with starts existing in the original Steph's corpus, so I'll have to match blog post tags and sentence starts, and if nothing fits, I'll just write "w00t" and hope that stephbot says something smart.

``` r
library("tidytext")
data("stop_words")

# get all possible start words
all_possible_starts <- markovifyR::generate_start_words(markov_model)
all_possible_starts <- dplyr::select(all_possible_starts, wordStart)
# for matching we'll use a version of these words
# that's all lower case, and without hashtags
all_possible_starts <- dplyr::mutate(all_possible_starts, 
                                     # escape encoding
                                     word = encodeString(wordStart),
                                     # lower case
                                     word = tolower(word)) %>%
                                     # from hashtag to real word
                           dplyr::filter(!stringr::str_detect(word, "#"))

get_corresponding_start <- function(tags, all_possible_starts){
  matches <- tibble::tibble(tag = tags) %>%
    tidytext::unnest_tokens(word, tag, token = "words") %>%
    # remove stop words
    dplyr::anti_join(stop_words, by = "word") %>%
    # close words
   fuzzyjoin::stringdist_left_join(all_possible_starts, 
                                  by = c("word"), 
                                  max_dist = 1, distance_col = "dist") %>%
    dplyr::filter(!is.na(wordStart)) %>%
    dplyr::filter(dist == min(dist))
  
  if(nrow(matches) == 0){
    "w00t"
  }else{
    set.seed(42)
    sample(matches$wordStart, 1)
  }
}

get_corresponding_start("data", all_possible_starts)
```

    ## [1] "Data"

``` r
get_corresponding_start("knitting", all_possible_starts)
```

    ## [1] "w00t"

The next step is to write an actual tweet text, with a start related to the topics, and with the URL to the blog post. One could also add hashtags based on the tags.

``` r
steph_describe_post <- function(post_df, markov_model, all_possible_starts){
  tags <- post_df$tag
  start <- get_corresponding_start(tags, all_possible_starts)
  set.seed(42)
  text <- steph_speak(start = start, markov_model = markov_model,
                      count =  1,
                      seed = sample(1:100, 1))
  tweet_text <- paste(text, post_df$url[1])
  tweet_text
}

all_posts <- split(blog_info, blog_info$url)
all_posts[[1]]$title[1]
```

    ## [1] "2016 PASS Board of Directors Candidate Town Halls"

``` r
all_posts[[1]]$tag
```

    ## [1] "Microsoft Data Platform" "Community"              
    ## [3] "pass"                    "elections"

``` r
steph_describe_post(all_posts[[1]], 
                    markov_model = markov_model,
                    all_possible_starts = all_possible_starts)
```

    ## [1] "PASS is gr8 for starting up in arms because it looks like a snow tan right? https://itsalocke.com/blog/2016-pass-board-of-directors-candidate-town-halls/"

``` r
all_posts[[77]]$title[1]
```

    ## [1] "satRday location voting now open"

``` r
all_posts[[77]]$tag
```

    ## [1] "user group"  "Community"   "r"           "R"           "speaking"   
    ## [6] "conferences" "satrday"

``` r
steph_describe_post(all_posts[[77]], 
                    markov_model = markov_model,
                    all_possible_starts = all_possible_starts)
```

    ## [1] "satRday location voting now open - be part of the tweets I used to send, follow someone as a community #rstats workshop with someone &amp; someone for some #rstats training in Excel/SQL/R/Python - this time - trying to limit thruput to increase ratio every yr &amp; struggle *a lot* - it's kind of thing, dunno bout skype https://itsalocke.com/blog/satrday-location-voting-now-open/"

Obviously in this version, `steph_describe_post` has the strength of sounding like Steph, but the weakness of *not* being informative. When reading this tweet I'd think "it's fun but not true."

Here is a way to quickly generate an informative but not fun and not necessarily Steph-like tweet using the `praise` package.

``` r
pr_describe_post <- function(post_df){
  tags <- toString(post_df$tag)
  url <- post_df$url[1]
  praise::praise(template = paste0("What a ${adjective} blog post about ", tags, "! ", url))
}
pr_describe_post(all_posts[[1]])
```

    ## [1] "What a beautiful blog post about Microsoft Data Platform, Community, pass, elections! https://itsalocke.com/blog/2016-pass-board-of-directors-candidate-town-halls/"

``` r
pr_describe_post(all_posts[[77]])
```

    ## [1] "What a premium blog post about user group, Community, r, R, speaking, conferences, satrday! https://itsalocke.com/blog/satrday-location-voting-now-open/"

Webshooting a blog post
=======================

Now that we have a method for getting a more or less informative and Steph-authentic tweet text, we need screenshots! Mara produces fantastic screenshots, have a look at [her Twitter feed](https://twitter.com/dataandme). She achieves this by being very good at screenshots and compositions of them, and also by reading posts to select what to screenshot. Here, my strategy will be to simply webshoot the regions under the post title and headers, hoping that they're representative of the blog post content and that they both offer a sort of summary and make go reading the whole post appealing. In order to make them look at list a bit pretty, I will use Locke Data official colours.

Here is the function to shot the region under title or a header, using `cliprect` to get part of the top of the post, and `expand` to get the area under a `selector`. Note, I seem to be having issues with PhantomJS, the web browser `webshot` uses, which means I do not get the fonts on the actual Locke Data website which is real pity.

``` r
shot_region <- function(df){
  header <- df$header
  number <- df$number
  url <- df$url
  post_name <- df$post_name
  if (header == ""){
    filename <- paste0("screenshot_tests/", post_name,
                                 number,
                                 "-title",".png")
    webshot::webshot(url = url,
                   cliprect = c(0, 0, 992, 200))
  }else{
    filename <- paste0("screenshot_tests/", post_name,
                                 number, "-",
                                 header,".png")
    webshot::webshot(url = url,
                   selector = paste0("#", header),
                   expand = c(5, 5, 200, 5))
  }
  magick::image_read("webshot.png") %>%
    magick::image_border(color = "#E8830C", geometry = "10x10") %>%
    magick::image_border(color = "#2165B6", geometry = "10x10") %>%
    magick::image_write(filename)
  
  file.remove("webshot.png")
}
```

Before using it one needs all headers from any post.

``` r
get_post_info <- function(post_df){
  url <- post_df$url[1]
  post_name <- stringr::str_replace(post_df$name[1],
                                     "\\.md", "")
  
  headers <-  httr::GET(url) %>%
     httr::content() %>%
    rvest::html_nodes("h2") %>%
    rvest::html_attrs() %>%
    unlist()
  
  tibble::tibble(url = url,
                 post_name = post_name,
                 header = c("", headers),
                 number = seq_along(header))
}

get_post_info(all_posts[[1]])
```

    ## # A tibble: 10 x 4
    ##    url                        post_name                header       number
    ##    <chr>                      <chr>                    <chr>         <int>
    ##  1 https://itsalocke.com/blo~ 2016-10-06-2016-pass-bo~ ""                1
    ##  2 https://itsalocke.com/blo~ 2016-10-06-2016-pass-bo~ allen-white       2
    ##  3 https://itsalocke.com/blo~ 2016-10-06-2016-pass-bo~ ben-miller        3
    ##  4 https://itsalocke.com/blo~ 2016-10-06-2016-pass-bo~ wendy-pastr~      4
    ##  5 https://itsalocke.com/blo~ 2016-10-06-2016-pass-bo~ eduardo-cas~      5
    ##  6 https://itsalocke.com/blo~ 2016-10-06-2016-pass-bo~ paresh-moti~      6
    ##  7 https://itsalocke.com/blo~ 2016-10-06-2016-pass-bo~ jason-strate      7
    ##  8 https://itsalocke.com/blo~ 2016-10-06-2016-pass-bo~ key-message~      8
    ##  9 https://itsalocke.com/blo~ 2016-10-06-2016-pass-bo~ who-am-i-li~      9
    ## 10 https://itsalocke.com/blo~ 2016-10-06-2016-pass-bo~ thoughts-on~     10

Now we can apply webshooting to get all screenshots for a post, and a bit of `magick` to make them prettier.

Here is how one would use the function for one post.

``` r
get_post_info(all_posts[[101]]) %>%
  split(.$header) %>%
  purrr::walk(shot_region) 
```

What would one tweet about this blog post by the way?

``` r
steph_describe_post(all_posts[[101]], 
                    markov_model = markov_model,
                    all_possible_starts = all_possible_starts)
```

    ## [1] "Agile is great for management looking to get back to Boston this year! https://itsalocke.com/blog/talking-data-and-docker/"

``` r
pr_describe_post(all_posts[[101]])
```

    ## [1] "What a beautiful blog post about Misc Technology, best practices, conferences, docker, open source, DataOps, agile! https://itsalocke.com/blog/talking-data-and-docker/"

Here is the result!

<blockquote class="twitter-tweet" data-lang="ca">
<p lang="en" dir="ltr">
🤖 What a beautiful blog post about Misc Technology, best practices, conferences, docker, open source, DataOps, agile! <a href="https://t.co/12rcKGXFNz">https://t.co/12rcKGXFNz</a>" <a href="https://t.co/1gVsl6zWeu">pic.twitter.com/1gVsl6zWeu</a>
</p>
— Maëlle Salmon 🐟 (@ma\_salmon) <a href="https://twitter.com/ma_salmon/status/963000763759898624?ref_src=twsrc%5Etfw">12 de febrer de 2018</a>
</blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
Apart from the font issue, it's not perfect because instead of having a strict size for the region under the title or header one should make it depend on the length of content inside each section.

Further work
============

So here it is, we can sort of maraaverickfy a blog post without reading it! The end result is not nearly as good as Mara's actual tweets, but hey, reading blog posts demands time! And brain power! Still, the most obvious improvement would be to really read Steph's posts, since they're good ones.

Now if I were to continue with my lazy approach, I'd first try to fix fonts on screenshots. I'd also like to add chibis to the screenshots, and maybe to use `rtweet` to build a real bot sending the tweets or a Shiny app combining both the text and screenshot generating functions, and the tags to plan tweeting regularly about all different (evergreen) topics covered on the blog. I also think the text-generating aspect could be explored further, and this way, maybe Steph could spend most of her time on a cruise without anyone's noticing her Twitter absence.
