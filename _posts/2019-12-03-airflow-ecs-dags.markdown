---
layout: post
title:  "Airflow ECS Dags!"
author: Frank van Lankvelt
date:   2019-12-03 12:50:56 +0100
categories: airflow aws
excerpt: Deploy an ECS TaskDefinition, run a DAG
---

# Data Products - from Analysis to Production
In recent years, Data Science has been grabbing a lot of attention in the news.
It appears that all new software that's being developed is powered by
Artificial Intelligence.  From Roombas vacuum cleaners that learn the layout
of your house to Deepminds AlphaGo program that's able to defeat the best human
players of the game of Go.  One can even buy a toothbrush that tells you how to
brush your teeth with the help of AI - [GENIUS X With Artificial Intelligence
Power Toothbrush](https://www.theverge.com/circuitbreaker/2019/10/25/20932250/oral-b-genius-x-connected-toothbrush-ai-artificial-intelligence).  So if everyone and their cat is doing it, how hard
can it be?

The technologies that power this burst of development have been developing
rapidly over the last few years.  There's Spark for data munging and
pre-processing, Jupyter for interactive analysis, Tensorflow/PyTorch for
image and language model development, Kafka for streaming, etcetera.  And these
are just some of the leading technologies, many of them have perfectly viable
alternatives.  

In order to make a data product, these technologies need to be strung together.
In addition, they must be joined with a general-purpose (micro)service,
database, etcetera.  In the words of Vicky Boykis, [We're still in the
steam-powered days of machine learning](https://vicki.substack.com/p/were-still-in-the-steam-powered-days).

## Airflow
One component that has gained a large market share is
[Airflow](https://airflow.apache.org),
a "platform to author, schedule and monitor workflows".  It enables data
engineers and scientists to manage their ETL (Extract, Load & Transform) needs
for obtaining data and the training jobs that turn this data into models.
Operations can be combined to form DAGs (Directed Acyclic Graphs), capturing
their mutual dependencies so that tasks are (only) carried out when the
necessary data is present.  These DAGs are typically scheduled to run daily,
with Airflow then making sure tasks are retried when they fail, logs are
captured for debugging purposes and alerts are sent when a DAG did not
complete succesfully.

A crucial point is that, like other monitoring and logging components, there is
only one instance of the Airflow service.  It contains the logic for all jobs
to be run, including their dependency graphs.  In addition, it needs to be able
to access all data sources and sinks and computing infrastructure to execute
jobs.

# Microservices and DevOps
Meanwhile agile service development has merged with operations in the
[you build it, you run it](https://aws.amazon.com/blogs/enterprise-strategy/enterprise-devops-why-you-should-run-what-you-build/) philosophy.
An agile team that develops a service is empowered to deploy it to production,
monitor its performance and logging.  It can thereby take responsibility and
quickly iterate to improve the service, balancing new functionality with 
improved resilience.  Central to this development is the capacity to be
self-sufficient, to be able to control all components (database, authorization,
message bus, caching etcetera).

The lead in this area has been taken by the cloud providers,
[Amazon Cloud Formation](https://aws.amazon.com/cloudformation/),
[Google Deployment Manager](https://cloud.google.com/deployment-manager/) and
[Azure Resource Manager](https://docs.microsoft.com/en-us/azure/azure-resource-manager/).  
Using these tools, it is possible to manage multiple resources (machines,
containers, storage, etcetera) as one logical unit.  This makes it possible
to spin up the complete infrastructure a service needs from a CI/CD pipeline.

# Data products at Royal FloraHolland
At [Royal FloraHolland](https://www.royalfloraholland.com/en),
the finest marketplace for flowers and plants for over one hundred years,
we are developing more and more data products.  Empowering growers with
real-time analytics in
[Floriday Insights](https://insights.floriday.com/en),
helping them set sensible prices in the
[Clock Presales](https://www.floriday.io/clockpresales),
assessing quality and consistency of batch photos, etcetera.

As the Data Science team, we try to blend in as much as we can with the
wider development landscape - consisting of [Floriday](https://www.floriday.io/en),
[Floramondo](https://www.royalfloraholland.com/floramondo) and
[FloraXchange](https://www.floraxchange.nl).  By deploying our services
on AWS, consolidating logs in [logzio](https://logz.io), gathering metrics
in [Cloudwatch Metrics](https://docs.aws.amazon.com/cloudwatch/index.html) and 
visualizing in [Grafana](https://grafana.com).  We develop using
[GitLab](https://about.gitlab.com) for CI/CD, engaging Cloud Formation to
manage resources - either with templates or the
[Cloud Development Kit](https://docs.aws.amazon.com/cdk/index.html).  
We're a bit peculiar in our preference for Python, but since the rest of the
Floriday is an eclectic mix of Java, Kotlin, .NET Core and Typescript, this is
hardly something that gets in the way of interoperability.

Each product is typically a separate GitLab project, with a Cloud Formation
stack for each the different AWS accounts we use.  Cloud formation manages the
ECS containers for services, the Lambda's for stream processing and
inter-account data transfer, the Sagemaker training jobs for model training,
Aurora Serverless for databases, Kinesis for real-time data transfer and S3 for
storage of training data and models.  An important concern is security, we only
allow access to resources when that's needed, using IAM roles and policies.
While developing the deployment infrastructure consists of always-too-slow
iterations, Cloud Formation generally works well for deployment of the
different components that make up a data product.

## The Airflow God
The one architectural component that breaks the project-product duality is
Airflow.  DAGs that logically and functionally belong to a project become part
of a different `airflow-dags` project.  In addition to hoarding code from each
of the data products that we have, `airflow-dags` forces them to agree on
dependencies.  Upgrading the DAG of one project suddenly forces one to upgrade
(and test!) all other DAGs that use the same (transitive) dependency.  Apart
from being a code magnet, the Airflow service needs to have access to all
resource that are accessed by any of the DAGs.  This makes it very easy for data
dependencies to hide - one project may unknowingly use the policies attached to
the Airflow service by another project to access data.

In object-oriented programming we would refer to Airflow as the
[*God Object*](https://en.wikipedia.org/wiki/God_object).

Airflow is very powerful, allowing dynamic generation of DAGs and able to talk
to a deluge of external services.  But truth be told, we do not need all of
this power.  I gladly sacrifice most of this power in preference of a solution
that keeps projects independent.

# Airflow as a service
What if we could treat DAGs just like another AWS resource type instead?
Each project would contain not just the code that is needed for an ETL or a
training job, but would be able to define the DAG, its tasks, scheduling,
alerting etcetera.  Deployment would be specified in the Cloud Formation
template for the AWS enviroment that executes the code.  And of course we
want scheduling, monitoring and alerting to be available from a single 
place - allowing inspection of the logs and to trigger jobs.

In order to do this, we need to do two things
- dynamically pick up DAGs from the ECS Task registry
- execute the tasks by triggering the ECS tasks

The code that we use to do this is available in the
[airflow-ecs-dags project](https://github.com/rfh-labs/airflow-ecs-dags).

## Executing tasks
In order to execute tasks, we need to launch an ECS task from an Airflow task.
Luckily, Airflow `contrib` already contains an
[ECS Operator](https://github.com/apache/airflow/blob/master/airflow/contrib/operators/ecs_operator.py).
The only thing missing for our purposes is the fetching of the logs - so that
we can read the logs in the Airflow user interface.

## Scanning for DAGs
This is the tricky part.  We need to encode the DAG (Directed Acyclic Graph)
in some way - preferably a way that's not too cumbersome to write or to digest.
Ideally, the DAG definition should be part of the "metadata" of an ECS Task.
Since there is no natural way to provide such additional, non-functional,
data we had to come up with our own.  Luckily a `ContainerDefinition` can
contain additional information in the form of `DockerLabels`.

# A Sample DAG
An ECS-Task-DAG in our projects now looks like this (in CloudFormation YAML
syntax):
```
  AirflowDag:
    Type: AWS::ECS::TaskDefinition
    Properties:
      ContainerDefinitions:
        - Cpu: 64
          Essential: true
          Image: "task-image:123-abcdefg"
          Memory: 1024
          MemoryReservation: 256
          Name: airflow-dag
          DockerLabels:
            airflow.dag.name: 'convert-it'
            airflow.dag.owner: 'Frank'
            airflow.dag.depends_on_past: 'false'
            airflow.dag.start_date: '2019-12-04T03:00:00'
            airflow.dag.email_on_failure: "true"
            airflow.dag.email_on_retry: "false"
            airflow.dag.concurrency: "12"
            airflow.dag.retries: "1"
            airflow.dag.email.0: 'Frank@rfh.example.com'
            airflow.dag.email.1: 'DataScience@rfh.example.com'
            airflow.tasks.latest_only.class: 'airflow.operators.latest_only_operator.LatestOnlyOperator'
            airflow.tasks.convert.args.0: '{{ ds }}'
            airflow.tasks.convert.depends.0: 'latest_only'
      Family: "airflow-train-dag"
      TaskRoleArn: !Ref AirflowDagTaskRole
```
This DAG consists of two tasks,
- `latest_only`, a standard Airflow operator.
- `convert`, a command on the container that takes Airflow's `execution date`
  as argument.  Note that this is specified using an Airflow macro.

The corresponding container has a simple `Dockerfile`:
```docker
FROM python:3.7
COPY . .
ENTRYPOINT ["python", "run.py"]
```

And a `python` file contains the actual logic:
```python
from datetime import datetime


def convert(run_date):
    dt = datetime.strptime(run_date, "%Y-%m-%d")
    print(f'Converted data of date {dt}')


if __name__ == '__main__':
    command = sys.argv[1]
    if command == 'convert':
        convert(run_date=sys.argv[2])
    else:
        raise Exception(f'Unknown command {command}')
```

# Limitations
Decoupling the task definition from Airflow introduces some limitations to what
we can do:
- sensors are not a natural part of this solution, as spinning up an ECS
  container feels too heavy for this kind of polling
- xcom - cross-task communication - would require making the output of a task
  available to other tasks.  We have no way to do that (yet?).
- DAG construction at runtime.  We use this functionality in Airflow to register
  the ECS DAGs, but have no way to make it available from an ECS Task itself.
- connection definitions from Airflow are not available, so an ECS Task needs
  to be provided the necessary connection details (hostname, credentials).

While these pieces of functionality certainly can be put to good use, they have
not presented unsurmountable challenges to us.  We were able to 
- join sensor tasks into the preceding tasks if that launched an external job
  (e.g. Glue, Sagemaker), or
- create common operators in the remaining skeleton `airflow-dags` project for
  those tasks (e.g. sensors) that were common to many DAGs
- drop `xcom` use.  It was not used a lot, we had no problem eliminating it
- specify connection defails with each DAG that needs them.  We see this as
  a positive change

# Conclusion
By moving our DAG definitions to the projects that needed them, we were able
to eliminate a lot of interdependencies between projects.  Deploying DAGs as
part of our (already existing) CloudFormation stacks also makes it easy to
synchronize changes to the DAG with those in the model definition (e.g.
Sagemaker), the service (ECS), storage (S3) and database (RDS) or any other
resource that is present in the stack.

Where previously we had periods of great instability when we were developing
operators, or upgrading dependencies, that were shared by multiple projects,
this now no longer occurs.

Be sure to check out the
[airflow-ecs-dags project](https://github.com/rfh-labs/airflow-ecs-dags)
and feel free to comment!

{% comment %} 

---
# Airflow
  - big ball of multi-project code
  - forces projects in lock-step
    - dependencies
    - deployment
  - deployment testing requires many environments

# Issues we ran into
  - gandalf
  - dependencies under heavy development

# Cloud deployment (AWS)
- aws model
  - each project has number of stacks
  - all architectural components are managed by the project

# Issues with cloud schedulers / workflow
- airflow UI
  - centralized logging
  - triggering jobs
  - backfill
- aws scheduling
  - bare bones

# 
- airflow on aws
  - use Airflow as a Service
  - each project develops independently
  - deploy ECS task as a dag
    - separate dependency management
    - separate deployment cycle
  - keep graph structure for task dependencies
  - pass (sagemaker/glue) logs through to airflow

{% endcomment %}
