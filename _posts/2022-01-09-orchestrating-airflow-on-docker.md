---
layout: post
title: "Running Airflow on Docker"
date: 2022-01-09 09:10:00 +0700
categories: experience
toc: true
comments: true
---
## Background
Before we begin to configure Airflow using Docker we must first understand why we need to use workflow orchestrator. Workflow orchestrator such as Airflow is used to programmatically author, schedule, and monitor workflows run for batch data processing. So why do we need to process the data? Well, at first if you start a company, an OLTP database will suffice. However, as the company grows you will need to engineer your products/services to attract more customer, you will need to do analysis in OLAP database. Now here's where workflow executor comes to the rescue.

### Why use Airflow?
The act of moving data from OLTP to OLAP database is called Extract-Transform-Load, or ETL for short. ETL process consists of extract phase, transformation phase, and loading phase; yeah.. just like the name. To execute an ETL job, we can use literally any scripting language; be it bash or Python. Usually when we define an ETL job, we want it to run in a specified schedule. So, you can use cron job to schedule the run natively without any additional workflow orchestrator. But one of Airflow's advantage is the ability to monitor all of your job runs in a single place. Oh your job run from yesterday is returning an error? You can look it up on Airflow and even restart it if needed. You want to execute a newly created job as if the date is last month? Well, with Airflow you can use backfill to run any job with specific execution date. Your machine cannot handle all of those data processing? With Airflow you can dedicate a machine as scheduler and webserver (for UI) and another separate machine as worker; and technically you can scale this worker nodes up if seems fit. Data teams has realized the strength of Airflow and therefore started moving from cron-based executor to Airflow ([more here](https://medium.com/videoamp/what-we-learned-migrating-off-cron-to-airflow-b391841a0da4)).

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

For now, ignore the `COPY` command there. It's basically used to copy file/folder to the image. I used `COPY` because the Crate library is a custom Python library and it isn't published to any public package repo. Therefore I have to copy it and install it using the full source code. The eight line purpose is exactly identical to the fifth line. The only difference is the library name.

Lastly, on the tenth line we change the user back to `airflow` in order to execute Airflow from the base image.

## Composing Dependencies
We have successfully modified Airflow base image previously using Dockerfile. While we can straight up run this image, we can make it much better. To execute Airflow, we need a database management system. By default, Airflow will use SQLite. However, we can attach PostgreSQL as Airflow backend for better performance.

But how to do that? Well, the answer is simple. We just need to compose another image with the previous Airflow image using Docker Compose. 

Still on `airflow` directory, create another file called `docker-compose.yaml`.

```
airflow
├── Dockerfile
└── docker-compose.yaml
```

Here is what my `docker-compose.yaml` file looks like. I have defined some common configuration and services that needs to be run along with Airflow.
<script src="https://gist.github.com/dion-ricky/dc0b9e10f14b404d239c4560a635a737.js"></script>

### Services
On line 33, you can see a list of defined services that will be run. Services defined on that files are: Postgres, Airflow Webserver, Airflow Scheduler, and lastly Airflow Init. On the next section I will explain each of those services in more details. 

On every service, we need to define some required and optional configuration. Configuration that is required on each service is image name (or build image from Dockerfile using `build: DOCKERFILE_PATH`). Every other configurations are optional or required specifically by the image.

#### Postgres
The first service defined is PostgreSQL as Airflow's backend database. As explained before, we need to configure the image name or build path. This service will use `postgres:13` or version 13 of the official PostgreSQL image from Docker Hub. We also need to configure the `environment` (on line 37) to make sure that we can connect to this database from Airflow using that specific credentials.

A database will need to have an access to file system right, but by default even if we didn't specify the `volumes` config, the service will run without problem. However for this case, I want to create a mapping from postgres data folder to `dbdata` directory. This is so that when I delete the whole composed service, I can keep the data untouched. The mapping is defined on line 41. How this works is kinda similar to creating symbolic link in Linux.

Next is healthcheck configuration on line 42 which can be omitted if you only use this in a non-production environment. This healthcheck will check that the service is alive every specific interval.

#### Airflow Webserver
Next we have Airflow Webserver service. On this composition, the Airflow job is separated into two service. This one is for the webserver, and the next service will act as both scheduler and worker.

Airflow webserver will provide UI to view, manage, and access all of Airflow's essential features. Here we use the common configuration with the name of `airflow-common` defined on top of the file, starting from line 5. That configuration will be using a `build` command and thus composing this service will first build the image from the given path.

We use common config here because we need to make sure that both Airflow webserver and scheduler are in sync and can communicate with each other correctly. On that common config, we also defined the volumes mapping as well.

However, take a look at line 28. Here we have a `depends_on` configuration that is used to make sure that the required service's is up and running.

#### Airflow Scheduler
Another responsibilities of Airflow are scheduler and worker. We will set this service to act as both scheduler and worker for Airflow. If you need to use other worker type, such as CeleryWorker to be able to scale the worker node, you can define it as a different service ([here is an example][celery]). In this project, I will be using LocalExecutor because there is no need for a scaling and I'm not running in a production environment.

On this service we will be using the same common configuration as Airflow Webserver service before. That is crucial to the successful deployment of this composition.

#### Airflow Init
This service will only need be executed once, technically only when we first created the composed container. However, Docker Compose will run this service every time we start the composed container. But that isn't a problem, because after that first initialization it will not overwrite or change anything.

This service is used to initialize Airflow database, kinda like creating new tables, preparing for the environment and stuff. I don't really know what's happening inside this `initdb` command. But one thing for sure is that, this command is needed before we can run any of the previously defined Airflow services.

## Starting the Container
Woah finally we are done with the configuration. Now to verify, make sure that your folder structure looks similar to this:
```
airflow
├── Dockerfile
└── docker-compose.yaml
```

To start the services from `docker-compose.yaml`, we need to run `docker-compose up`. But, remember, we have to initialize Airflow database before running their service. So first we have to run the `airflow-init` and `postgres` service. To do that, we simply need to run `docker-compose up airflow-init`. Just pass the service name after the `up` argument. You will see something similar to this after executing that command:
```
Starting future_postgres_1 ... done
Starting future_airflow-init_1 ... done
Attaching to future_airflow-init_1
airflow-init_1       | DB_BACKEND=postgresql+psycopg2
airflow-init_1       | DB_HOST=postgres
airflow-init_1       | DB_PORT=5432
airflow-init_1       | 
...
airflow-init_1       | Done.
future_airflow-init_1 exited with code 0
```

Please keep in mind that your compose project name will most likely be different from mine.

After the initialization finished you can run `docker-compose up` to start all the other service that are not started from initialization previously. To see the list of running container you can execute `docker ps -a`. If you have successfully deployed all of the services, you will see something like this:
```
CONTAINER ID   IMAGE                                  COMMAND                  CREATED       STATUS                             PORTS                                       NAMES
b367c199b89e   future_airflow-webserver               "/usr/bin/dumb-init …"   2 days ago    Up 54 seconds (healthy)            0.0.0.0:8090->8080/tcp, :::8090->8080/tcp   future_airflow-webserver_1
7c8d7480c0e0   future_airflow-scheduler               "/usr/bin/dumb-init …"   2 days ago    Up 54 seconds (health: starting)   8080/tcp                                    future_airflow-scheduler_1
508f5db30625   postgres:13                            "docker-entrypoint.s…"   2 days ago    Up 3 minutes (healthy)             5432/tcp                                    future_postgres_1
```

## Evaluating System Usage
Now this step is optional but doing this will help you to reduce your system resource usage caused by running two separate instance of Airflow. Now here's the deal, Docker by itself is pretty chunky. You needs at least 4GB of memory to be allocated to Docker alone, ideally 8GB (from Airflow's documentation).

If you use an official `docker-compose.yaml` from Airflow ([here][airflow-compose]), the minimum requirements are:
- CPU: 2 cores
- Memory: 4 GB
- Disk space: 10 GB

If you take a look at my `docker-compose.yaml` previously on [composing dependencies][my-compose] section, you will notice that in the `environment` configuration part, I have included several scheduler and webserver env variables.
```
environment:
    ...
    AIRFLOW__SCHEDULER__MIN_FILE_PROCESS_INTERVAL: 60
    AIRFLOW__WEBSERVER__WORKERS: 2
    AIRFLOW__WEBSERVER__WORKER_REFRESH_INTERVAL: 1800
    AIRFLOW__WEBSERVER__WEB_SERVER_WORKER_TIMEOUT: 300
```

`min_file_process_interval`, according to the documentation is an interval for Airflow to parse DAG file. The default value is 30, which means that every 30 seconds, Airflow will rescan your DAG folder to parse—import libraries, possibly run init on each operator, and check for changes in task execution order—all of the DAG definition that you have. Which is crazily expensive, in my opinion. Now, while parsing the DAG file, Airflow will still try to do it's job to run the DAG. But if you set this number too low, then Airflow will have a hard time executing your DAG.

`webserver__workers`, this number is the number of workers to run the Gunicorn web server. I set this to 2 since the only one that have access to this webserver is me and one other person in my team. If you have dozens of data engineers accessing the webserver, you might want to increase this number.

`webserver__worker_refresh_interval` and `webserver__web_server_worker_timeout` didn't do much impact on your resource, it's basically used to make sure that your webserver worker is running well. And if it encountered a bottleneck somewhere, Airflow will wait for the timeout interval before timing out on that worker. While worker refresh interval is number of seconds to wait before refreshing a batch of workers.

## Conclusion
Runnning ETL without Airflow or any workflow executor is technically possible. But if you as a data engineer wants to make use of Airflow's features to easily manage all of your workflows, then why not. While configuring Airflow can be a headache, especially if you are not familiar with infrastructures and networking. Fear not, as Docker will come to help you to run Airflow anywhere using the exact same configuration that you only need to wrote once.

However keep in mind the resource usage of the Airflow, and if possible try to scale your worker nodes up to be able to run more ETL jobs. That's it for this blog, if you do have any question feel free to join the discussion on the bottom of this page.

## References
Kisller, E. (2021, November 8). A Beginner’s Guide to Understanding and Building Docker Images. JFrog. [https://jfrog.com/knowledge-base/a-beginners-guide-to-understanding-and-building-docker-images/][1]. \[[Archived][1-a]\].

Apache Airflow Documentation — Airflow Documentation. (n.d.). Apache Airflow Documentation. Retrieved January 9, 2022, from [https://airflow.apache.org/docs/apache-airflow/1.10.12/][airflow-docs] and [https://airflow.readthedocs.io/en/1.10.12/][airflow-readthedocs].

[1]: https://jfrog.com/knowledge-base/a-beginners-guide-to-understanding-and-building-docker-images/
[1-a]: https://web.archive.org/web/20220109092402/https://jfrog.com/knowledge-base/a-beginners-guide-to-understanding-and-building-docker-images/

[airflow-docs]: https://airflow.apache.org/docs/apache-airflow/1.10.12/
[airflow-readthedocs]: https://airflow.readthedocs.io/en/1.10.12/

[celery]: https://gist.github.com/dion-ricky/35652e7a1fd196dc1a90e24495bd968d
[my-compose]: #composing-dependencies
[airflow-compose]: https://airflow.apache.org/docs/apache-airflow/stable/docker-compose.yaml