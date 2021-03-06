---
title: Markdown based web analytics? Rectangle your blog
author: maelle
type: post
date: "2018-02-21T13:22:40+00:00"
spacious_page_layout:
  - default_layout
categories:
  - Data Science
  - R
tags:
  - r

---


Locke Data's great blog is Markdown-based. What this means is that all blog posts exist as Markdown files: you can see all of them [here](https://github.com/lockedatapublished/blog/tree/master/content/posts). They then get rendered to html by [some sort of magic cough `blogdown` cough](https://github.com/rstudio/blogdown) we don't need to fully understand here. For marketing efforts, I needed a census of existing blog posts along with some precious information. Here is how I got it, in other words here is how I [*rectangled*](https://speakerdeck.com/jennybc/data-rectangling) the website GitHub repo and live version to serve our needs.

Note: This should be applicable to any Markdown-based blog!

What are you about, dear blog posts?
====================================

To find out what a blog post is about, I read its tags and categories, that live in the YAML header of each post, see [for instance this one](https://github.com/lockedatapublished/blog/blob/master/content/posts/2013-05-19-time-to-go-home.md). Just a note, thank you, [participants in this Stack Overflow thread](https://stackoverflow.com/questions/25022016/get-all-file-names-from-a-github-repo-through-the-github-api).

Getting all blog posts names and path
-------------------------------------

I used the [`gh` package](https://github.com/r-lib/gh) to interact with GitHub V3 API.

``` r
# get link to all posts and their filename
posts <- gh::gh("/repos/:owner/:repo/contents/:path", 
                owner = "lockedatapublished",
                repo = "blog",
                path = "content/posts")

gh_posts <- tibble::tibble(name = purrr::map_chr(posts, "name"),
                           path = purrr::map_chr(posts, "path"),
                           raw = purrr::map_chr(posts, "download_url"))
```

Here is the table I got:

``` r
library("magrittr")
gh_posts %>%
  head() %>%
  knitr::kable()
```

| name                                                             | path                                                                           | raw                                                                                                                                               |
|:-----------------------------------------------------------------|:-------------------------------------------------------------------------------|:--------------------------------------------------------------------------------------------------------------------------------------------------|
| 2013-05-11-setting-up-wordpress-on-azure.md                      | content/posts/2013-05-11-setting-up-wordpress-on-azure.md                      | <https://raw.githubusercontent.com/lockedatapublished/blog/master/content/posts/2013-05-11-setting-up-wordpress-on-azure.md>                      |
| 2013-05-12-objectless-chec-boxes-using-vba.md                    | content/posts/2013-05-12-objectless-chec-boxes-using-vba.md                    | <https://raw.githubusercontent.com/lockedatapublished/blog/master/content/posts/2013-05-12-objectless-chec-boxes-using-vba.md>                    |
| 2013-05-19-time-to-go-home.md                                    | content/posts/2013-05-19-time-to-go-home.md                                    | <https://raw.githubusercontent.com/lockedatapublished/blog/master/content/posts/2013-05-19-time-to-go-home.md>                                    |
| 2013-05-24-center-across-selection.md                            | content/posts/2013-05-24-center-across-selection.md                            | <https://raw.githubusercontent.com/lockedatapublished/blog/master/content/posts/2013-05-24-center-across-selection.md>                            |
| 2013-05-25-making-charts-with-conditionally-coloured-series.md   | content/posts/2013-05-25-making-charts-with-conditionally-coloured-series.md   | <https://raw.githubusercontent.com/lockedatapublished/blog/master/content/posts/2013-05-25-making-charts-with-conditionally-coloured-series.md>   |
| 2013-05-29-synchronising-schema-between-mssql-mysql-with-ssis.md | content/posts/2013-05-29-synchronising-schema-between-mssql-mysql-with-ssis.md | <https://raw.githubusercontent.com/lockedatapublished/blog/master/content/posts/2013-05-29-synchronising-schema-between-mssql-mysql-with-ssis.md> |

There are 169 posts in this table.

Getting all blog post image links
---------------------------------

In a blogdown blog, you do not need to be consistent with image naming, as long as you give the correct link inside your post. Images used on Steph's blog live [here](https://github.com/lockedatapublished/blog/tree/master/static/img) and their names often reflect the blog post name, but not always. I thought it could be useful to have a table of all blog posts images. I wrote a function that downloads the content of each post and extract image links.

``` r
get_pics <- function(path){
    message(path)
    file <- gh::gh("/repos/:owner/:repo/contents/:path", 
                   owner = "lockedatapublished",
                   repo = "blog",
                   path = path)
    content <- rawToChar(base64enc::base64decode(file$content))
    # get links to imgs
    img <- stringr::str_match(content, 'src=\\\".*?\\/img\\/(.*?)\"' )[,2]
    tibble::tibble(path = path,
                   img = img)
}
```

Let's see what it does for one path.

``` r
get_pics("content/posts/2013-05-11-setting-up-wordpress-on-azure.md") %>%
  knitr::kable()
```

| path                                                      | img                                  |
|:----------------------------------------------------------|:-------------------------------------|
| content/posts/2013-05-11-setting-up-wordpress-on-azure.md | azurescreenshot1\_ujw0yl\_finrxl.png |

Ok then I simply needed to apply it to all posts.

``` r
pics <- purrr::map_df(gh_posts$path, get_pics)
gh_pics <- dplyr::left_join(gh_posts, pics, by = "path")
gh_pics <- dplyr::filter(gh_pics, !is.na(img))
readr::write_csv(gh_pics, path = "data/gh_imgs.csv")
```

Having this table, one could run some analysis of the number of images by post, extract pictures when promoting a post, and tidy a website. I think Steph's filenames are good, but I could imagine renaming files based on the blog post they appear in if it had not been done previously (and changing the link inside posts obviously), but hey why clean if one can link the data anyway.

Getting all tags and categories
-------------------------------

The code here is similar to the previous one but slightly more complex because I wrote the post content inside a temporary .yaml file in order to read it using `rmarkdown::yaml_front_matter`.

``` r
get_one_yaml <- function(path){
  print(path)
  file <- gh::gh("/repos/:owner/:repo/contents/:path", 
                 owner = "lockedatapublished",
                 repo = "blog",
                 path = path)
  content <- rawToChar(base64enc::base64decode(file$content))
  
  # the yaml function didn't like this
  content <- stringr::str_replace_all(content, "â€œ", "")
  content <- stringr::str_replace_all(content, "â€\u009d", "")
  write(content, "temporary.md")
  data <- rmarkdown::yaml_front_matter("temporary.md")
  file.remove("temporary.md")
  
  # is this an elegant solution? No :-)
  # but this way I'll get both categories and tags
  # and won't get issues if several categories/tags
  categories <- data$categories
  data$categories <- NULL
  data <- dplyr::as_data_frame(data)
  
  
  if("tags" %in% names(data)){
    
    data <- dplyr::mutate(data, value = TRUE)
    data <- tidyr::spread(data, tags, value, fill = FALSE)
    data <- dplyr::mutate(data, path = path)
  }
  
  
  if(!is.null(categories)){
    categories <- dplyr::tibble(categories = paste0("cat_", categories))
    categories <- dplyr::mutate(categories, value = TRUE)
    categories <- tidyr::spread(categories, categories, value, fill = FALSE)
  }
  
  data <- cbind(data, categories)
  data
}
```

I'll illustrate this with one post:

``` r
get_one_yaml("content/posts/2013-05-11-setting-up-wordpress-on-azure.md") %>%
  knitr::kable()
```

    ## [1] "content/posts/2013-05-11-setting-up-wordpress-on-azure.md"

| title                         | author | type | date                      |  dsq\_thread\_id| azure | blogging | ec2  | wordpress | path                                                      | cat\_Misc Technology |
|:------------------------------|:-------|:-----|:--------------------------|----------------:|:------|:---------|:-----|:----------|:----------------------------------------------------------|:---------------------|
| Setting up WordPress on Azure | Steph  | post | 2013-05-11T22:14:43+00:00 |               NA| TRUE  | TRUE     | TRUE | TRUE      | content/posts/2013-05-11-setting-up-wordpress-on-azure.md | TRUE                 |

I then used the function over all paths.

``` r
info <- purrr::map_df(gh_posts$path, get_one_yaml)
gh_posts <- dplyr::left_join(gh_posts, info, by = "path")

readr::write_csv(gh_posts, path = "data/gh_posts.csv")
```

Tags and categories can be useful to perform an action on blog posts depending on them (e.g., make a list of all posts related to X), and to analyse the use of tags and categories: what topics did Steph blog about over time? Coupled with traffic data, what topics are the most read?

So, I know a lot about blog posts now, but if I were to say read or webshoot them, where should I go?

Where do you live, dear blog posts?
===================================

Often, the URL of a blog post can be guessed based on its title, e.g. [this one](https://raw.githubusercontent.com/lockedatapublished/blog/master/content/posts/2013-05-19-time-to-go-home.md) can be read [here](https://itsalocke.com/blog/time-to-go-home/). But even if the transition from the Markdown file information to an URL is logical, it was best to get URLs from the in situ blog posts, and then join them to the blog post information collected previously, since some special characters got special treatment that I could not fully understand by looking at `blogdown` source code.

I first extracted all posts URLs from the website map.

``` r
library("magrittr")

# get links and tags
sitemap <- xml2::read_xml("https://itsalocke.com/blog/sitemap.xml") %>% 
  xml2::as_list() %>%
  .$urlset

# probably re-inventing the wheel
get_one <- function(element, what){
  one <- unlist(element[[what]])
  if(is.null(one)){
    one <- ""
  }
  
  one
}

# tibble with everything
sitemap <- tibble::tibble(url = purrr::map_chr(sitemap, get_one, "loc"),
                       date = purrr::map_chr(sitemap, get_one, "lastmod"))

# only blog posts
blog <- dplyr::filter(sitemap, !stringr::str_detect(url, "tags\\/"))
blog <- dplyr::filter(blog, !stringr::str_detect(url, "categories\\/"))
blog <- dplyr::filter(blog, !stringr::str_detect(url, "statuses\\/"))
blog <- dplyr::filter(blog, url != "https://itsalocke.com/blog/stuff-i-read-this-week/")
blog <- dplyr::filter(blog, !stringr::str_detect(url, "https://itsalocke.com/blog/.*?\\/.*?\\/"))
blog <- dplyr::filter(blog, url != "https://itsalocke.com/blog/")
blog <- dplyr::filter(blog, url != "https://itsalocke.com/blog/posts/")
```

This is the resulting "sitemap".

``` r
head(blog) %>%
  knitr::kable()
```

| url                                                                                    | date                      |
|:---------------------------------------------------------------------------------------|:--------------------------|
| <https://itsalocke.com/blog/how-to-maraaverickfy-a-blog-post-without-even-reading-it/> | 2018-02-12T11:49:40+00:00 |
| <https://itsalocke.com/blog/connecting-to-sql-server-on-shinyapps.io/>                 | 2018-01-31T09:29:42+00:00 |
| <https://itsalocke.com/blog/year-2-of-locke-data/>                                     | 2018-01-29T00:00:00+00:00 |
| <https://itsalocke.com/blog/working-with-pdfs---scraping-the-pass-budget/>             | 2017-12-29T00:00:00+00:00 |
| <https://itsalocke.com/blog/using-blogdown-with-an-existing-hugo-site/>                | 2017-12-20T00:00:00+00:00 |
| <https://itsalocke.com/blog/data-manipulation-in-r/>                                   | 2017-12-18T21:29:42+00:00 |

Now how do I join it to the data previously collected? I first tried to reproduce what `blogdown` does to post titles. Note that in some cases Steph had to had a "slug" by hand when migrating her blog to blogdown, which is what I use when it's available.

``` r
gh_info <- readr::read_csv("data/gh_posts.csv")
gh_info <- dplyr::filter(gh_info, !stringr::str_detect(name, "\\.Rmd"))

unique(gh_info$slug)
```

    ## [1] NA                                          
    ## [2] "satrdays-voting-closes-may-31st"           
    ## [3] "my-pass-summit2016-submissions-feedback"   
    ## [4] "using-blogdown-with-an-existing-hugo-site" 
    ## [5] "working-with-pdfs-scraping-the-pass-budget"

``` r
# https://github.com/rstudio/blogdown/blob/0c4c30dbfb3ae77b27594685902873d63c2894ad/R/utils.R#L277
dash_filename = function(string, pattern = '[^[:alnum:]^\\.]+') {
  tolower(string) %>%
    stringr::str_replace_all("â", "") %>%
    stringr::str_replace_all("DataOps.*? it.*?s a thing (honest)",
                             "dataops--its-a-thing-honest") %>%
    stringr::str_replace_all(pattern, '-') %>%
    stringr::str_replace_all('^-+|-+$', '') 
    
}
gh_info <- dplyr::mutate(gh_info, 
                         base = ifelse(!is.na(slug), slug, title),
                         base = dash_filename(base),
                         false_url = paste0("https://itsalocke.com/blog/", 
                                      base, "/"))
```

Here are a few "false URLs" that I get. They're often the right URLs, but not always!

``` r
tail(gh_info$false_url)
```

    ## [1] "https://itsalocke.com/blog/data-manipulation-in-r/"                                  
    ## [2] "https://itsalocke.com/blog/using-blogdown-with-an-existing-hugo-site/"               
    ## [3] "https://itsalocke.com/blog/working-with-pdfs-scraping-the-pass-budget/"              
    ## [4] "https://itsalocke.com/blog/year-2-of-locke-data/"                                    
    ## [5] "https://itsalocke.com/blog/connecting-to-sql-server-on-shinyapps.io/"                
    ## [6] "https://itsalocke.com/blog/how-to-maraaverickfy-a-blog-post-without-even-reading-it/"

The cases in which they're not the URL are often cases with double dashes for instance. In order to be quick, I decided to simply join them using string distance, because the false and right URLs will be quite similar anyway.

``` r
all_info <- fuzzyjoin::stringdist_left_join(blog, gh_info, 
                                            by = c("url" = "false_url"),
                                            max_dist = 3)
all_info$url[(is.na(all_info$raw))]
```

    ## [1] "https://itsalocke.com/blog/being-an-organised-sponsor-sce-p3/"

``` r
all_info$title[duplicated(all_info$raw)]
```

    ## [1] "Shiny module design patterns: Pass module input to other modules" 
    ## [2] "Shiny module design patterns: Pass module inputs to other modules"
    ## [3] "optiRum 0.37.1 now out"                                           
    ## [4] "optiRum 0.37.3 now out"

``` r
readr::write_csv(all_info, path = "data/all_info_about_posts.csv")
```

So what are the posts that did not get mapped properly, in brief?

-   two very close announcements of a new version of `optiRum`. I can correct that by hand, but since URLs are needed to webshoot evergreen posts, I will probably ignore them.

-   two very close blog post titles about Shiny that I shall correct.

A taste of the usefulness of such data!
=======================================

But hey here is what one gets from the website!

``` r
head(all_info) %>%
  knitr::kable()
```

| url                                                                                    | date.x                    | name                                                     | path                                                                   | raw                                                                                                                                       | title                                                    | author | type | date.y              | dsq\_thread\_id | azure | blogging | ec2 | wordpress | cat\_Misc Technology | check boxes | Excel | macros | tick | vba | merge | ribbon | chart | datawarehouse | mssql | mysql | ssis | cat\_Microsoft Data Platform | format | presentation | sql server | statuspost | user group | tutorial | cat\_Community | marketing | sqlrelay | windows | analysis | r    | cat\_R | presentations | SSRS | blog | best practices | continuous integration | ssas | unit testing | quick tip | sql fundamentals | code | hacks | lookup | VB  | speaking | socialauthorbio\_custom\_checkbox\_meta | data analysis | r basics | web scraping | cat\_Data Science | presenting | photoshop | productivity | knitr | rmarkdown | conferences | tip | mentoring | professional development | magrittr | software development in r | stuff I read this week | reports | shiny | docker | security | microsoft | business | data visualisation | open source | twitter | git | httr | source control | tfs | tfsR | visual studio online | visual studio team services | cat\_DataOps | machine learning | api | blob storage | stream analytics | editing | managing a team | software development | data.table | fonts | visual studio | zoomit | gini coefficient | logistic regression | optiRum | statistics | latex | test coverage | travis-ci | agile | azure data factory | auto deploying r documentation | documentation | github | spacious\_page\_layout | dataops | dlm | wdt | wit | mango | Community | satrday | enclosure | anchor model | data modelling | medrianchor | sixth normal form | r consortium | dell | icons | resolution | scaling | text | xps13 | surviving | business intelligence | microsoft edge | pdf | linux from windows | pageant | plink | putty | ssh | ssh tunnel | powerbi | sql tricks | sqlcardiff | haveibeenpwned | hibpwned | mockaroo | shiny design patterns | odbc | slug                                       | azureml | chocolatey | censornet | data breaches | powershell | feedback | pass | experts | diversity | elitism | aws | azure automation | etl | failures | lightning talks | novalite\_template | sponsor | sponsoring community events | sponsorship basics | asteroids | game | gamemaker | cat\_Uncategorized | sql relay | elections | ssl | ssdt | slack | css | hugo | call for contributors | gwdp | opportunity | bash | linux | scripts | attendee | mvp | mvp summit | azure functions | data science | data mining | process | time series | python | sentiment analysis | cardiff | logistic regressions | video | magick | rocker | rstudio | training | get started | coveralls | troubleshooting | cran | datasauRus | linear regression | bot services | bots | qnamaker | skype | microsoft r server | temporal tables | executive briefing | microsoft cognitive services | purrr | image                | fundamentals | data manipulation | data wrangling | tidyverse | blogdown | Locke-Data | freetds | shinyapps | base                                                     | false\_url                                                                             |
|:---------------------------------------------------------------------------------------|:--------------------------|:---------------------------------------------------------|:-----------------------------------------------------------------------|:------------------------------------------------------------------------------------------------------------------------------------------|:---------------------------------------------------------|:-------|:-----|:--------------------|:----------------|:------|:---------|:----|:----------|:---------------------|:------------|:------|:-------|:-----|:----|:------|:-------|:------|:--------------|:------|:------|:-----|:-----------------------------|:-------|:-------------|:-----------|:-----------|:-----------|:---------|:---------------|:----------|:---------|:--------|:---------|:-----|:-------|:--------------|:-----|:-----|:---------------|:-----------------------|:-----|:-------------|:----------|:-----------------|:-----|:------|:-------|:----|:---------|:----------------------------------------|:--------------|:---------|:-------------|:------------------|:-----------|:----------|:-------------|:------|:----------|:------------|:----|:----------|:-------------------------|:---------|:--------------------------|:-----------------------|:--------|:------|:-------|:---------|:----------|:---------|:-------------------|:------------|:--------|:----|:-----|:---------------|:----|:-----|:---------------------|:----------------------------|:-------------|:-----------------|:----|:-------------|:-----------------|:--------|:----------------|:---------------------|:-----------|:------|:--------------|:-------|:-----------------|:--------------------|:--------|:-----------|:------|:--------------|:----------|:------|:-------------------|:-------------------------------|:--------------|:-------|:-----------------------|:--------|:----|:----|:----|:------|:----------|:--------|:----------|:-------------|:---------------|:------------|:------------------|:-------------|:-----|:------|:-----------|:--------|:-----|:------|:----------|:----------------------|:---------------|:----|:-------------------|:--------|:------|:------|:----|:-----------|:--------|:-----------|:-----------|:---------------|:---------|:---------|:----------------------|:-----|:-------------------------------------------|:--------|:-----------|:----------|:--------------|:-----------|:---------|:-----|:--------|:----------|:--------|:----|:-----------------|:----|:---------|:----------------|:-------------------|:--------|:----------------------------|:-------------------|:----------|:-----|:----------|:-------------------|:----------|:----------|:----|:-----|:------|:----|:-----|:----------------------|:-----|:------------|:-----|:------|:--------|:---------|:----|:-----------|:----------------|:-------------|:------------|:--------|:------------|:-------|:-------------------|:--------|:---------------------|:------|:-------|:-------|:--------|:---------|:------------|:----------|:----------------|:-----|:-----------|:------------------|:-------------|:-----|:---------|:------|:-------------------|:----------------|:-------------------|:-----------------------------|:------|:---------------------|:-------------|:------------------|:---------------|:----------|:---------|:-----------|:--------|:----------|:---------------------------------------------------------|:---------------------------------------------------------------------------------------|
| <https://itsalocke.com/blog/how-to-maraaverickfy-a-blog-post-without-even-reading-it/> | 2018-02-12T11:49:40+00:00 | 2018-02-12-maraaverickfyer.md                            | content/posts/2018-02-12-maraaverickfyer.md                            | <https://raw.githubusercontent.com/lockedatapublished/blog/master/content/posts/2018-02-12-maraaverickfyer.md>                            | How to maraaverickfy a blog post without even reading it | maelle | post | 2018-02-12 11:49:40 | NA              | NA    | NA       | NA  | NA        | NA                   | NA          | NA    | NA     | NA   | NA  | NA    | NA     | NA    | NA            | NA    | NA    | NA   | NA                           | NA     | NA           | NA         | NA         | NA         | NA       | NA             | NA        | NA       | NA      | NA       | TRUE | TRUE   | NA            | NA   | NA   | NA             | NA                     | NA   | NA           | NA        | NA               | NA   | NA    | NA     | NA  | NA       | NA                                      | NA            | NA       | NA           | TRUE              | NA         | NA        | NA           | NA    | NA        | NA          | NA  | NA        | NA                       | NA       | NA                        | NA                     | NA      | NA    | NA     | NA       | NA        | NA       | NA                 | NA          | NA      | NA  | NA   | NA             | NA  | NA   | NA                   | NA                          | NA           | NA               | NA  | NA           | NA               | NA      | NA              | NA                   | NA         | NA    | NA            | NA     | NA               | NA                  | NA      | NA         | NA    | NA            | NA        | NA    | NA                 | NA                             | NA            | NA     | default\_layout        | NA      | NA  | NA  | NA  | NA    | NA        | NA      | NA        | NA           | NA             | NA          | NA                | NA           | NA   | NA    | NA         | NA      | NA   | NA    | NA        | NA                    | NA             | NA  | NA                 | NA      | NA    | NA    | NA  | NA         | NA      | NA         | NA         | NA             | NA       | NA       | NA                    | NA   | NA                                         | NA      | NA         | NA        | NA            | NA         | NA       | NA   | NA      | NA        | NA      | NA  | NA               | NA  | NA       | NA              | NA                 | NA      | NA                          | NA                 | NA        | NA   | NA        | NA                 | NA        | NA        | NA  | NA   | NA    | NA  | NA   | NA                    | NA   | NA          | NA   | NA    | NA      | NA       | NA  | NA         | NA              | NA           | NA          | NA      | NA          | NA     | NA                 | NA      | NA                   | NA    | NA     | NA     | NA      | NA       | NA          | NA        | NA              | NA   | NA         | NA                | NA           | NA   | NA       | NA    | NA                 | NA              | NA                 | NA                           | NA    | NA                   | NA           | NA                | NA             | NA        | NA       | NA         | NA      | NA        | how-to-maraaverickfy-a-blog-post-without-even-reading-it | <https://itsalocke.com/blog/how-to-maraaverickfy-a-blog-post-without-even-reading-it/> |
| <https://itsalocke.com/blog/connecting-to-sql-server-on-shinyapps.io/>                 | 2018-01-31T09:29:42+00:00 | 2018-01-31-freetds.md                                    | content/posts/2018-01-31-freetds.md                                    | <https://raw.githubusercontent.com/lockedatapublished/blog/master/content/posts/2018-01-31-freetds.md>                                    | Connecting to SQL Server on shinyapps.io                 | steph  | post | 2018-01-31 09:29:42 | NA              | NA    | NA       | NA  | NA        | NA                   | NA          | NA    | NA     | NA   | NA  | NA    | NA     | NA    | NA            | NA    | NA    | NA   | NA                           | NA     | NA           | TRUE       | NA         | NA         | NA       | NA             | NA        | NA       | NA      | NA       | TRUE | TRUE   | NA            | NA   | NA   | NA             | NA                     | NA   | NA           | NA        | NA               | NA   | NA    | NA     | NA  | NA       | NA                                      | NA            | NA       | NA           | NA                | NA         | NA        | NA           | NA    | NA        | NA          | NA  | NA        | NA                       | NA       | NA                        | NA                     | NA      | NA    | NA     | NA       | NA        | NA       | NA                 | NA          | NA      | NA  | NA   | NA             | NA  | NA   | NA                   | NA                          | NA           | NA               | NA  | NA           | NA               | NA      | NA              | NA                   | NA         | NA    | NA            | NA     | NA               | NA                  | NA      | NA         | NA    | NA            | NA        | NA    | NA                 | NA                             | NA            | NA     | default\_layout        | NA      | NA  | NA  | NA  | NA    | NA        | NA      | NA        | NA           | NA             | NA          | NA                | NA           | NA   | NA    | NA         | NA      | NA   | NA    | NA        | NA                    | NA             | NA  | NA                 | NA      | NA    | NA    | NA  | NA         | NA      | NA         | NA         | NA             | NA       | NA       | NA                    | NA   | NA                                         | NA      | NA         | NA        | NA            | NA         | NA       | NA   | NA      | NA        | NA      | NA  | NA               | NA  | NA       | NA              | NA                 | NA      | NA                          | NA                 | NA        | NA   | NA        | NA                 | NA        | NA        | NA  | NA   | NA    | NA  | NA   | NA                    | NA   | NA          | NA   | NA    | NA      | NA       | NA  | NA         | NA              | NA           | NA          | NA      | NA          | NA     | NA                 | NA      | NA                   | NA    | NA     | NA     | NA      | NA       | NA          | NA        | NA              | NA   | NA         | NA                | NA           | NA   | NA       | NA    | NA                 | NA              | NA                 | NA                           | NA    | img/WorkingWithR.png | NA           | NA                | NA             | NA        | NA       | NA         | TRUE    | TRUE      | connecting-to-sql-server-on-shinyapps.io                 | <https://itsalocke.com/blog/connecting-to-sql-server-on-shinyapps.io/>                 |
| <https://itsalocke.com/blog/year-2-of-locke-data/>                                     | 2018-01-29T00:00:00+00:00 | 2018-01-29-locke-data-update.md                          | content/posts/2018-01-29-locke-data-update.md                          | <https://raw.githubusercontent.com/lockedatapublished/blog/master/content/posts/2018-01-29-locke-data-update.md>                          | Year 2 of Locke Data                                     | Steph  | NA   | 2018-01-29 00:00:00 | NA              | NA    | NA       | NA  | NA        | NA                   | NA          | NA    | NA     | NA   | NA  | NA    | NA     | NA    | NA            | NA    | NA    | NA   | NA                           | NA     | NA           | NA         | NA         | NA         | NA       | NA             | NA        | NA       | NA      | NA       | NA   | TRUE   | NA            | NA   | NA   | NA             | NA                     | NA   | NA           | NA        | NA               | NA   | NA    | NA     | NA  | NA       | NA                                      | NA            | NA       | NA           | NA                | NA         | NA        | NA           | NA    | NA        | NA          | NA  | NA        | NA                       | NA       | NA                        | NA                     | NA      | NA    | NA     | NA       | NA        | NA       | NA                 | NA          | NA      | NA  | NA   | NA             | NA  | NA   | NA                   | NA                          | NA           | NA               | NA  | NA           | NA               | NA      | NA              | NA                   | NA         | NA    | NA            | NA     | NA               | NA                  | NA      | NA         | NA    | NA            | NA        | NA    | NA                 | NA                             | NA            | NA     | NA                     | NA      | NA  | NA  | NA  | NA    | NA        | NA      | NA        | NA           | NA             | NA          | NA                | NA           | NA   | NA    | NA         | NA      | NA   | NA    | NA        | NA                    | NA             | NA  | NA                 | NA      | NA    | NA    | NA  | NA         | NA      | NA         | NA         | NA             | NA       | NA       | NA                    | NA   | NA                                         | NA      | NA         | NA        | NA            | NA         | NA       | NA   | NA      | NA        | NA      | NA  | NA               | NA  | NA       | NA              | NA                 | NA      | NA                          | NA                 | NA        | NA   | NA        | NA                 | NA        | NA        | NA  | NA   | NA    | NA  | NA   | NA                    | NA   | NA          | NA   | NA    | NA      | NA       | NA  | NA         | NA              | NA           | NA          | NA      | NA          | NA     | NA                 | NA      | NA                   | NA    | NA     | NA     | NA      | NA       | NA          | NA        | NA              | NA   | NA         | NA                | NA           | NA   | NA       | NA    | NA                 | NA              | NA                 | NA                           | NA    | NA                   | NA           | NA                | NA             | NA        | NA       | TRUE       | NA      | NA        | year-2-of-locke-data                                     | <https://itsalocke.com/blog/year-2-of-locke-data/>                                     |
| <https://itsalocke.com/blog/working-with-pdfs---scraping-the-pass-budget/>             | 2017-12-29T00:00:00+00:00 | 2017-12-29-working-with-pdfs-scraping-the-pass-budget.md | content/posts/2017-12-29-working-with-pdfs-scraping-the-pass-budget.md | <https://raw.githubusercontent.com/lockedatapublished/blog/master/content/posts/2017-12-29-working-with-pdfs-scraping-the-pass-budget.md> | Working with PDFs - scraping the PASS budget             | Steph  | NA   | 2017-12-29 00:00:00 | NA              | NA    | NA       | NA  | NA        | NA                   | NA          | NA    | NA     | NA   | NA  | NA    | NA     | NA    | NA            | NA    | NA    | NA   | NA                           | NA     | NA           | NA         | NA         | NA         | NA       | NA             | NA        | NA       | NA      | NA       | NA   | TRUE   | NA            | NA   | NA   | NA             | NA                     | NA   | NA           | NA        | NA               | NA   | NA    | NA     | NA  | NA       | NA                                      | NA            | NA       | NA           | TRUE              | NA         | NA        | NA           | NA    | NA        | TRUE        | NA  | NA        | NA                       | NA       | NA                        | NA                     | NA      | NA    | NA     | NA       | NA        | NA       | NA                 | NA          | NA      | NA  | NA   | NA             | NA  | NA   | NA                   | NA                          | NA           | NA               | NA  | NA           | NA               | NA      | NA              | NA                   | NA         | NA    | NA            | NA     | NA               | NA                  | NA      | NA         | NA    | NA            | NA        | NA    | NA                 | NA                             | NA            | NA     | NA                     | NA      | NA  | NA  | NA  | NA    | TRUE      | NA      | NA        | NA           | NA             | NA          | NA                | NA           | NA   | NA    | NA         | NA      | NA   | NA    | NA        | NA                    | NA             | NA  | NA                 | NA      | NA    | NA    | NA  | NA         | NA      | NA         | NA         | NA             | NA       | NA       | NA                    | NA   | working-with-pdfs-scraping-the-pass-budget | NA      | NA         | NA        | NA            | NA         | NA       | TRUE | NA      | NA        | NA      | NA  | NA               | NA  | NA       | NA              | NA                 | NA      | NA                          | NA                 | NA        | NA   | NA        | NA                 | NA        | NA        | NA  | NA   | NA    | NA  | NA   | NA                    | NA   | NA          | NA   | NA    | NA      | NA       | NA  | NA         | NA              | NA           | NA          | NA      | NA          | NA     | NA                 | NA      | NA                   | NA    | NA     | NA     | NA      | NA       | NA          | NA        | NA              | NA   | NA         | NA                | NA           | NA   | NA       | NA    | NA                 | NA              | NA                 | NA                           | NA    | NA                   | NA           | NA                | NA             | NA        | NA       | NA         | NA      | NA        | working-with-pdfs-scraping-the-pass-budget               | <https://itsalocke.com/blog/working-with-pdfs-scraping-the-pass-budget/>               |
| <https://itsalocke.com/blog/using-blogdown-with-an-existing-hugo-site/>                | 2017-12-20T00:00:00+00:00 | 2017-12-20-using-blogdown-with-an-existing-hugo-site.md  | content/posts/2017-12-20-using-blogdown-with-an-existing-hugo-site.md  | <https://raw.githubusercontent.com/lockedatapublished/blog/master/content/posts/2017-12-20-using-blogdown-with-an-existing-hugo-site.md>  | Using blogdown with an existing Hugo site                | steph  | NA   | 2017-12-20 00:00:00 | NA              | NA    | TRUE     | NA  | NA        | TRUE                 | NA          | NA    | NA     | NA   | NA  | NA    | NA     | NA    | NA            | NA    | NA    | NA   | NA                           | NA     | NA           | NA         | NA         | NA         | NA       | TRUE           | NA        | NA       | NA      | NA       | NA   | TRUE   | NA            | NA   | TRUE | NA             | NA                     | NA   | NA           | NA        | NA               | NA   | NA    | NA     | NA  | NA       | NA                                      | NA            | NA       | NA           | NA                | NA         | NA        | NA           | NA    | NA        | NA          | NA  | NA        | NA                       | NA       | NA                        | NA                     | NA      | NA    | NA     | NA       | NA        | NA       | NA                 | NA          | NA      | NA  | NA   | NA             | NA  | NA   | NA                   | NA                          | NA           | NA               | NA  | NA           | NA               | NA      | NA              | NA                   | NA         | NA    | NA            | NA     | NA               | NA                  | NA      | NA         | NA    | NA            | NA        | NA    | NA                 | NA                             | NA            | NA     | NA                     | NA      | NA  | NA  | NA  | NA    | NA        | NA      | NA        | NA           | NA             | NA          | NA                | NA           | NA   | NA    | NA         | NA      | NA   | NA    | NA        | NA                    | NA             | NA  | NA                 | NA      | NA    | NA    | NA  | NA         | NA      | NA         | NA         | NA             | NA       | NA       | NA                    | NA   | using-blogdown-with-an-existing-hugo-site  | NA      | NA         | NA        | NA            | NA         | NA       | NA   | NA      | NA        | NA      | NA  | NA               | NA  | NA       | NA              | NA                 | NA      | NA                          | NA                 | NA        | NA   | NA        | NA                 | NA        | NA        | NA  | NA   | NA    | NA  | TRUE | NA                    | NA   | NA          | NA   | NA    | NA      | NA       | NA  | NA         | NA              | NA           | NA          | NA      | NA          | NA     | NA                 | NA      | NA                   | NA    | NA     | NA     | NA      | NA       | NA          | NA        | NA              | NA   | NA         | NA                | NA           | NA   | NA       | NA    | NA                 | NA              | NA                 | NA                           | NA    | NA                   | NA           | NA                | NA             | NA        | TRUE     | NA         | NA      | NA        | using-blogdown-with-an-existing-hugo-site                | <https://itsalocke.com/blog/using-blogdown-with-an-existing-hugo-site/>                |
| <https://itsalocke.com/blog/data-manipulation-in-r/>                                   | 2017-12-18T21:29:42+00:00 | 2017-12-18-datamanipulationinr.md                        | content/posts/2017-12-18-datamanipulationinr.md                        | <https://raw.githubusercontent.com/lockedatapublished/blog/master/content/posts/2017-12-18-datamanipulationinr.md>                        | Data Manipulation in R                                   | steph  | post | 2017-12-18 21:29:42 | NA              | NA    | NA       | NA  | NA        | NA                   | NA          | NA    | NA     | NA   | NA  | NA    | NA     | NA    | NA            | NA    | NA    | NA   | NA                           | NA     | NA           | NA         | NA         | NA         | NA       | NA             | NA        | NA       | NA      | NA       | TRUE | TRUE   | NA            | NA   | NA   | NA             | NA                     | NA   | NA           | NA        | NA               | NA   | NA    | NA     | NA  | NA       | NA                                      | NA            | NA       | NA           | NA                | NA         | NA        | NA           | NA    | NA        | NA          | NA  | NA        | NA                       | NA       | NA                        | NA                     | NA      | NA    | NA     | NA       | NA        | NA       | NA                 | NA          | NA      | NA  | NA   | NA             | NA  | NA   | NA                   | NA                          | NA           | NA               | NA  | NA           | NA               | NA      | NA              | NA                   | NA         | NA    | NA            | NA     | NA               | NA                  | NA      | NA         | NA    | NA            | NA        | NA    | NA                 | NA                             | NA            | NA     | default\_layout        | NA      | NA  | NA  | NA  | NA    | TRUE      | NA      | NA        | NA           | NA             | NA          | NA                | NA           | NA   | NA    | NA         | NA      | NA   | NA    | NA        | NA                    | NA             | NA  | NA                 | NA      | NA    | NA    | NA  | NA         | NA      | NA         | NA         | NA             | NA       | NA       | NA                    | NA   | NA                                         | NA      | NA         | NA        | NA            | NA         | NA       | NA   | NA      | NA        | NA      | NA  | NA               | NA  | NA       | NA              | NA                 | NA      | NA                          | NA                 | NA        | NA   | NA        | NA                 | NA        | NA        | NA  | NA   | NA    | NA  | NA   | NA                    | NA   | NA          | NA   | NA    | NA      | NA       | NA  | NA         | NA              | NA           | NA          | NA      | NA          | NA     | NA                 | NA      | NA                   | NA    | NA     | NA     | NA      | NA       | NA          | NA        | NA              | NA   | NA         | NA                | NA           | NA   | NA       | NA    | NA                 | NA              | NA                 | NA                           | NA    | NA                   | TRUE         | TRUE              | TRUE           | TRUE      | NA       | NA         | NA      | NA        | data-manipulation-in-r                                   | <https://itsalocke.com/blog/data-manipulation-in-r/>                                   |

When were blog posts published?

``` r
library("ggplot2")
all_info <- dplyr::mutate(all_info, date = anytime::anytime(date.x))
ggplot(all_info) +
  geom_point(aes(date), y = 0.5, col = "#2165B6", size = 0.9) +
  hrbrthemes::theme_ipsum(grid = "Y") 
```

![](../img/rectangle-your-blog1.png)

So posting, from this crude viz, look fairly regular with a few gaps.

One could also look at the categories.

``` r
all_info <- dplyr::select(all_info, - base, - false_url)
categories_info <- all_info %>%
  tidyr::gather("category", "value", 11:ncol(all_info)) %>%
  dplyr::filter(!is.na(value)) %>%
  dplyr::filter(stringr::str_detect(category, "cat\\_")) %>%
  dplyr::mutate(category = stringr::str_replace(category, "cat\\_", ""))
categories <- categories_info %>%
  dplyr::count(category, sort = TRUE) 

knitr::kable(categories)
```

| category                |    n|
|:------------------------|----:|
| R                       |   89|
| Community               |   61|
| Microsoft Data Platform |   42|
| Data Science            |   36|
| Misc Technology         |   29|
| DataOps                 |   25|
| Uncategorized           |    3|

And when where these categories used?

``` r
categories_info <- dplyr::mutate(categories_info, date = anytime::anytime(date.x))
ggplot(categories_info) +
  geom_point(aes(date, category), col = "#2165B6", size = 0.9) +
  hrbrthemes::theme_ipsum(grid = "Y")
```

![](../img/rectangle-your-blog2.png)

In the most recent period, R and Data Science seem to be getting more love than the other categories.

Let's see what other/more exciting things we can do with this data, to help make Locke Data blog even better and more read!
