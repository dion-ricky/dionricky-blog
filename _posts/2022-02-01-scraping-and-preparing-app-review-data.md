---
layout: post
title: "Scraping App Review Data for Analysis"
date: 2022-02-01 13:40:00 +0700
categories: data data-engineer
toc: true
comments: true
---
## Background
Mobile or portable devices are the most widely used technology of this era. With the increase of internet coverage and the pandemic situation, people are getting used to staying indoors and limiting their outdoor activities. This results in increased usage of mobile devices in addition to increased online commerce transactions. Users are free to download and install any e-commerce app to their liking. Their choice, though personal and subjective, were influenced by the players of the e-commerce sector. Businesses are competing to provide better, cheaper, and safer shopping experiences. However, that competition produces an astronomically long list of options for users to choose from. While that's just a natural implication of competition, users are overwhelmed by those choices. They need a comparison, a simple score, or others' experience<sup>1</sup> to quickly decide which one is the most suitable for their needs.

Users will look at the app reviews to compare and get general understanding of each app strengths and weaknesses. Businesses can also analyze their reviews to make improvement on their services and resolve problems on their internal business processes. Therefore, review data is very important.

In this post, I will explore the collection and cleaning process of App Reviews data. We will start from the data collection step, followed by the preparation and lastly we will show how to automate this data pipeline using Airflow.

## Data Collection
There are two main sources of app reviews, Google Play Store and Apple Store. We will refer to Google Play Store as Play Store from now on, since it is a bit too much to repeat it again and again.

For scraping Play Store reviews, I use [google-play-store][jomingyu-scraper] library by JoMingyu. It's a great scraper in general. It supports pausing using continuation, supports scraping app details and even app permissions, also the author is still actively maintaining it so it's another plus for this library.

For Apple Store however, things are a bit bleak. Since my pipeline uses Python, I'm forced to look for scraping library that are written in Python. At the time I work on this project, I couldn't find any decent scraper. The best one that I've found is written in JavaScript. Since I'm running out of time, I decided to just bite the bullet. I constructed an automated daily scraper using [app-store-scraper][facu-scraper] library by facundoolano. The automated scraper is running on a Cloud Function instance in Google Cloud Platform. The construction of this function however, is not included in this post.

### Play Store Review
Here is an example on how to scrape review data from a game called Penguin Isle.
```python
from google_play_scraper import Sort, reviews

result, continuation_token = reviews(
    'com.fantome.penguinisle',
    lang='en',
    country='us',
    sort=Sort.MOST_RELEVANT,
    count=3,
    filter_score_with=5
)
```

The reviews from google-play-store library contains a lot of information. Let's see what the output of the previous command is.
```python
[
    {
        "userName": "Alyssa Williams",
        "userImage": "https://lh3.googleusercontent.com/-cVEHKr7mzv8/AAAAAAAAAAI/AAAAAAAAAAA/AKF05nB2r3GUkji31m0tC4ylFNiVMpmNWA/photo.jpg",
        "content": "This is literally the best idle game I have ever played. The penguins waddle around and live their best lives in the cutest little outfits. I just unlocked the little penguins and I have been sobbing uncontrollably for ten minutes because they are so adorable. There are only two suggestions I have for this game: more of the penguin info ads. I love them. I have learned so much about all the teeny fellas. Secondly, I would like to be able to name my 'guins so I can tell them apart.",
        "score": 5,
        "thumbsUpCount": 54,
        "reviewCreatedVersion": "1.16",
        "at": datetime.datetime(2020, 2, 24, 17, 19, 34),
        "replyContent": "Hello, We will gradually improve the various systems in the game to enhance the player's game experience. We have recorded your suggestions and feedback to the planner. If you have any other suggestions and ideas, please feel free to contact us at penguinisle@habby.com.Thank you for playing!",
        "repliedAt": datetime.datetime(2020, 2, 24, 18, 30, 42),
        "reviewId": "gp:AOqpTOE0Iy5S9Je1F8W1BgCl6l_TCFP_QN4qGtRATX3PeB5VV9aZu6UHfMWdYFF1at4qZ59xxLNHFqYLql5SL-k"
    },
    ...
]
```

As you can see, straight from the output the data is pretty detailed already. I'm pretty satisfied with it, so we will continue to the App Store scraping section next.

### App Store Review
As I've explained before, the construction of the automated scraper function is not going to be explained in this post. Therefore, I can only show you the scraping result. Here's what it looks like.
```python
[
    {
        "id": "8301510944",
        "userName": "Sonyabelle",
        "userUrl": "https://itunes.apple.com/id/reviews/id200106552",
        "version": "2.165.0",
        "score": 4,
        "title": "Tokped terbaik",
        "text": "Tokped keren! Beli yoga matt gratis krn dpt point!",
        "url": "https://itunes.apple.com/id/review?id=1001394201&type=Purple%20Software",
        "updated": "2022-01-30T16:55:08-07:00"
    },
    ...
]
```

Not much information is captured by this scraper, but I'll need to make do with it somehow. Now that we have data from two sources, we can't possibly just merge them as once because of data type and schema issues. So, in the next section I will explain the preparation process for the review data.

## Data Preparation
Since the review data are scraped from different source, aside from cleaning it we need to transform it as well. We will make sure the schema is uniform and are fit for those data. Here's the schema for the staging data layer.
```
column_name             data_type        mode
=============================================
app_name                   STRING    NULLABLE
_____________________________________________
app_id                     STRING    NULLABLE
_____________________________________________
review_id                  STRING    NULLABLE
_____________________________________________
user_name                  STRING    NULLABLE
_____________________________________________
user_image                 STRING    NULLABLE
_____________________________________________
review                     STRING    NULLABLE
_____________________________________________
rating                      INT64    NULLABLE
_____________________________________________
thumbs_up_count             INT64    NULLABLE
_____________________________________________
app_version                STRING    NULLABLE
_____________________________________________
created_date            TIMESTAMP    NULLABLE
_____________________________________________
reply                      STRING    NULLABLE
_____________________________________________
replied_at              TIMESTAMP    NULLABLE
_____________________________________________
platform                   STRING    NULLABLE
_____________________________________________
dag_execution_date      TIMESTAMP    NULLABLE
_____________________________________________
```

For the cleaning part, we need to make sure that the data are not duplicate. To prevent duplicate data, we can do a deduplication by the review id. Next, we need to coalesce null data with empty state of the data type. For example, null string is replaced with empty string, null integer is replaced with 0, and so on.

### App Details Data
Aside from review data, I also collect the app details containing their version, recent changes, description, etc. They're scraped and processed using the same method as review data. However, since their columns is too long to show, I will only put the staging layer schema here.

Here's how their cleaned structure look like.
```
column_name             data_type        mode
=============================================
app_id                     STRING    NULLABLE
_____________________________________________
alt_app_id                 STRING    NULLABLE
_____________________________________________
app_name                   STRING    NULLABLE
_____________________________________________
url                        STRING    NULLABLE
_____________________________________________
version                    STRING    NULLABLE
_____________________________________________
description                STRING    NULLABLE
_____________________________________________
platform                   STRING    NULLABLE
_____________________________________________
genre_id                   STRING    NULLABLE
_____________________________________________
genre                      STRING    NULLABLE
_____________________________________________
rating                    FLOAT64    NULLABLE
_____________________________________________
rating_count                INT64    NULLABLE
_____________________________________________
review_count                INT64    NULLABLE
_____________________________________________
original_price            FLOAT64    NULLABLE
_____________________________________________
is_on_sale                BOOLEAN    NULLABLE
_____________________________________________
is_free                   BOOLEAN    NULLABLE
_____________________________________________
currency                   STRING    NULLABLE
_____________________________________________
size_in_byte              FLOAT64    NULLABLE
_____________________________________________
min_os_version             STRING    NULLABLE
_____________________________________________
released_date           TIMESTAMP    NULLABLE
_____________________________________________
updated_date            TIMESTAMP    NULLABLE
_____________________________________________
screenshots                STRING       ARRAY
_____________________________________________
recent_changes             STRING    NULLABLE
_____________________________________________
developer                  STRING    NULLABLE
_____________________________________________
developer_id               STRING    NULLABLE
_____________________________________________
developer_email            STRING    NULLABLE
_____________________________________________
developer_website          STRING    NULLABLE
_____________________________________________
content_rating             STRING    NULLABLE
_____________________________________________
dag_execution_date      TIMESTAMP    NULLABLE
_____________________________________________
```

## Summary
The steps of collection app review data are scraping and cleaning. App reviews data are scraped from Google Play Store using [google-play-scraper][jomingyu-scraper] and Apple Store using [app-store-scraper][facu-scraper]. Aside from reviews, I also scraped app details data that contains the description, app version, recent changes, etc; to enrich the review data for analysis.

Huge thanks to the author of the scraper library, [JoMingyu][jomingyu] and [facundoolano][facu].

## References
\[1\] Genc-Nayebi, N., & Abran, A. (2017). A systematic literature review: Opinion mining studies from mobile app store user reviews. Journal of Systems and Software, 125, 207–219. [https://doi.org/10.1016/j.jss.2016.11.027](https://doi.org/10.1016/j.jss.2016.11.027)

[jomingyu-scraper]: https://github.com/JoMingyu/google-play-scraper/
[jomingyu]: https://github.com/JoMingyu
[facu-scraper]: https://github.com/facundoolano/app-store-scraper
[facu]: https://github.com/facundoolano
[my-airflow]: https://dionricky.com/tech/data-engineer/2022/01/09/running-airflow-in-docker.html