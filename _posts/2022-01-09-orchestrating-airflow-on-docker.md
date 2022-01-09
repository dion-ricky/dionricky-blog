---
layout: post
title: "Running Airflow on Docker"
date: 2022-01-09 09:10:00 +0700
categories: experience
toc: true
---
## Table of contents
{:.no_toc}

* Table of contents
{:toc}

## Background
Before we begin to configure Airflow using Docker we must first understand why we need to use workflow orchestrator. Workflow orchestrator such as Airflow is used to programmatically author, schedule, and monitor workflows run for batch data processing. So why do we need to process the data? Well, at first if you start a company, an OLTP database will suffice. However, as the company grows you will need to engineer your products/services to attract more customer, you will need to do analysis in OLAP database. Now here's where workflow executor comes to the rescue.

### Why use Airflow?
The act of moving data from OLTP to OLAP database is called Extract-Transform-Load, or ETL for short. ETL process consists of extract phase, transformation phase, and loading phase; yeah,, just like the name. To execute an ETL job, we can use literally any scripting language; be it bash or Python. Usually when we define an ETL job, we want it to run in a specified schedule. So, you can use cron job to schedule the run natively without any additional workflow orchestrator. But one of Airflow's advantage is the ability to monitor all of your job runs in a single place. Oh your job run from yesterday is returning an error? You can look it up on Airflow and even restart it if needed. You want to execute a newly created job as if the date is last month? Well, with Airflow you can use backfill to run any job with specific execution date. Your machine cannot handle all of those data processing? With Airflow you can dedicate a machine as scheduler and webserver (for UI) and another separate machine as worker; and technically you can scale this worker nodes up if seems fit. Data teams has realized the strength of Airflow and therefore started moving from cron-based executor to Airflow ([more here](https://medium.com/videoamp/what-we-learned-migrating-off-cron-to-airflow-b391841a0da4)).

### Why running Airflow on Docker?
You can, pretty easily, install Airflow through pip (Python dependency manager) but there's a lot of configuration to do upfront. Not to mention the pain in configuring worker node networks if you want to scale it up using another machine. If you have to repeat the configuration on every machine, it will get very tedious really soon. Don't forget that your machine might have different OS distro, version; and maybe even different Python version, which will further complicate the installation process. Luckily, we can make use of Docker containerization technology to wrap our 'app' and make sure it will run on the exact same environment using a single configuration file. Thus, on this journey we will try to create a working containerized Airflow using Docker.

## Modifying the Image
Apache has provided Airflow base image on Docker Hub [repository](https://hub.docker.com/r/apache/airflow). While this image is enough for us to start using Airflow, we will most likely need to add more dependency in the near future. So why not make it future proof while we're at it.

Now assuming you are in 'airflow' directory create a file called 'Dockerfile'. After that, make sure your directory matches this structure:

```
airflow
└── Dockerfile
```

Inside this Dockerfile we will modify Apache's Airflow base image to add additional Python dependencies. The snippet below shows the modified base image with two additional dependency, Google Play Scraper and App Store Scraper library. You can add your own dependency using the exact same command `RUN pip install xyz` and replace `xyz` with the dependency name.

<script src="https://gist.github.com/dion-ricky/71855424c0becf53f61431eee07b2ae5.js"></script>

### Dissecting the Layers
The first line on the snippet shows `FROM apache/airflow:1.10.12-python3.6`. That line of code basically says, use image with the name `apache/airflow` and having tag `1.10.12-python3.6`. Image name is unlikely to change, unless you want to use other image. However, tag is like version or flavor indicator. You can look image tag up on the image repo in the Tags tab, or [visit here](https://hub.docker.com/r/apache/airflow/tags). This line will create the first layer on the, what will be build, Docker image.

> Each of the lines that make up a Docker image is known as **layer**. These layers form a series of intermediate images, built one on top of the other in stages, where each layer is dependent on the layer immediately below it. ... Thus, you should organize layers that change most often as high up in the stack as possible. This is because, when you make changes to a layer in your image, Docker not only rebuilds that particular layer, but all layers built from it. Therefore, a change to a layer at the top of a stack involves the least amount of computational work to rebuild the entire image. [(Kisller, E., 2021)][1]

Moving on to the next line, on the third line we have `USER root`. This `USER` command changes current user, just like `su - username` command on bash. Why this line is needed? Because to install Python dependencies using pip, we need permission to write to `/usr/local/bin`. If we don't have that permission, when we then try to run the lines below, they will return a `Permission denied` error.

Next, on the fifth line we have the `RUN` command that executes a shell command. The command `pip install google-play-scraper` will install Google Play Scraper Python library. You can use any command here that can be executed on shell.

For now, ignore the `COPY` command there. It's basically used to copy file/folder to the image. I used `COPY` because the App Store Scraper is a custom Python library and it isn't published to any Python package repo. Therefore I have to copy it and install it using the full source code. The seventh line purpose is exactly identical to the fifth line. The only difference is the library name that is specified.

Lastly, on the ninth line we change the user back to `airflow` in order to execute Airflow from the base image.

## Composing Dependencies
We have successfully modified Airflow base image previously using Dockerfile. While we can straight up run this image, we can make it much better. To execute Airflow, we need a database management system. By default, Airflow will use SQLite. However, we can attach PostgreSQL as Airflow backend for better performance.

But how to do that? Well, the answer is simple. We just need to compose another image with the previous Airflow image using Docker Compose. 

Still on `airflow` directory, create another file called `docker-compose.yaml`.

```
airflow
├── Dockerfile
└── docker-compose.yaml
```

<script src="https://gist.github.com/dion-ricky/dc0b9e10f14b404d239c4560a635a737.js"></script>

### Services

#### Postgres

#### Airflow Webserver

#### Airflow Scheduler

#### Airflow Init

## Starting the Container

## Evaluating CPU Usage

## Conclusion

## References
[(Kisller, E., 2021)][1-a]

[1]: https://jfrog.com/knowledge-base/a-beginners-guide-to-understanding-and-building-docker-images/
[1-a]: https://web.archive.org/web/20220109092402/https://jfrog.com/knowledge-base/a-beginners-guide-to-understanding-and-building-docker-images/