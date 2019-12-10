---
layout: post
title:  "Airflow ECS Dags!"
author: Frank van Lankvelt
date:   2019-12-03 12:50:56 +0100
categories: airflow aws
excerpt: Deploy an ECS TaskDefinition, run a DAG
---

## Data Products - from Analysis to Production
The technologies that power the burst of AI development have been developing
rapidly over the last few years.  In order to make a data product, these
technologies need to be strung together.  In addition, they must be joined with
a general-purpose (micro)service, database, etcetera.  In the words of Vicky
Boykis, [We're still in the steam-powered days of machine
learning](https://vicki.substack.com/p/were-still-in-the-steam-powered-days).

At Royal FloraHolland, we're building a variety of data products.  When doing
this, like other data science teams, we're confronted with the job to join
two worlds - that of *data* and that of *services*.

### Data - Airflow
The component that's specific to data products, in addition to those used for
microservices in general, is [Airflow](https://airflow.apache.org).  Like other
monitoring and logging components, there is only one instance of the Airflow
service.  It contains the logic for all jobs to be run, including their
dependency graphs.  In addition, it needs to be able to access all data sources
and sinks and computing infrastructure to execute jobs.

### Services - Microservices and DevOps
An agile team that develops a service is empowered to deploy it to production
and monitor its performance and logging.  The
[you build it, you run it](https://aws.amazon.com/blogs/enterprise-strategy/enterprise-devops-why-you-should-run-what-you-build/)
philosophy allows quick iterations, balancing new functionality with 
improved resilience.  Central to this development is the capacity to be
self-sufficient, to be able to control all components (database, authorization,
message bus, caching etcetera).

## Data products at Royal FloraHolland
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
on AWS, consolidating logs in [logz.io](https://logz.io), gathering metrics
in [Cloudwatch Metrics](https://docs.aws.amazon.com/cloudwatch/index.html) and 
visualizing in [Grafana](https://grafana.com).  We develop using
[GitLab](https://about.gitlab.com) for CI/CD, engaging CloudFormation to
manage resources - either with templates or the
[Cloud Development Kit](https://docs.aws.amazon.com/cdk/index.html).  
We're a bit peculiar in our preference for Python, but since the rest of the
Floriday is an eclectic mix of Java, Kotlin, .NET Core and Typescript, this is
hardly something that gets in the way of interoperability.

Each product is typically a separate GitLab project, with a CloudFormation
stack for each the different AWS accounts we use.  CloudFormation manages the
ECS containers for services, the Lambdas for stream processing and
inter-account data transfer, the Sagemaker training jobs for model training,
Aurora Serverless for databases, Kinesis for real-time data transfer and S3 for
storage of data and models.  For security, we only
allow access to resources when that's needed, using IAM roles and policies.
While developing the deployment infrastructure consists of always-too-slow
iterations, CloudFormation generally works well for deployment of the
different components that make up a data product.

### The Airflow God
The one architectural component that breaks the project-product duality is
Airflow.  DAGs that logically and functionally belong to a project become part
of a different `airflow-dags` project.  In addition to hoarding code from each
of the data products that we have, `airflow-dags` forces them to agree on
dependencies.  Upgrading the DAG of one project suddenly forces one to upgrade
(and test!) all other DAGs that use the same (transitive) dependency.  Apart
from being a code magnet, the Airflow service needs to have access to all
resources that are accessed by any of the DAGs.  This makes it very easy for data
dependencies to hide - one project may unknowingly use the policies attached to
the Airflow service by another project to access data.

In object-oriented programming we would refer to Airflow as the
[*God Object*](https://en.wikipedia.org/wiki/God_object).

Airflow is very powerful, allowing dynamic generation of DAGs and able to talk
to a deluge of external services.  But truth be told, we do not need all of
this power.  I gladly sacrifice most of this power in preference of a solution
that keeps projects independent.

## Airflow as a service
Imaging that we could treat DAGs just like another AWS resource type.
Each project would contain not just the code that is needed for an ETL or a
training job, but would be able to define the DAG, its tasks, scheduling,
alerting etcetera.  Deployment would be specified in the CloudFormation
template for the AWS enviroment that executes the code.  And of course we
want scheduling, monitoring and alerting to be available from a single 
place - allowing inspection of the logs and to trigger jobs.

In order to do this, we need to do two things:
- dynamically pick up DAGs from the ECS Task registry,
- execute the tasks by triggering the ECS tasks.

The code that we use to do this is available in the
[airflow-ecs-dags project](https://github.com/rfh-labs/airflow-ecs-dags).

In order to execute tasks, we need to launch an ECS task from an Airflow task.
Luckily, Airflow `contrib` already contains an
[ECS Operator](https://github.com/apache/airflow/blob/master/airflow/contrib/operators/ecs_operator.py).
The only thing missing for our purposes is the fetching of the logs - so that
we can read the logs in the Airflow user interface.

### Scanning for DAGs
This is the tricky part.  We need to encode the DAG (Directed Acyclic Graph)
in some way - preferably a way that's not too cumbersome to write or to digest.
Ideally, the DAG definition should be part of the "metadata" of an ECS Task.
Since there is no natural way to provide such additional, non-functional,
data we had to come up with our own.  Luckily a `ContainerDefinition` can
contain additional information in the form of `DockerLabels`.

## A Sample DAG
An ECS-Task-DAG in our projects now looks like this (in CloudFormation YAML
syntax):
```
{% raw %}
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
{% endraw %}
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

## Limitations
Decoupling the task definition from Airflow introduces some limitations to what
we can do:
- Having a sensor as a separate task is not a natural part of this solution, as
  spinning up an ECS container feels too heavy for this kind of polling.
- XCom - cross-task communication - would require making the output of a task
  available to other tasks.  We have no way to do that (yet?).
- DAG construction at runtime.  We use this functionality in Airflow to register
  the ECS DAGs, but have no way to make it available from an ECS Task itself.
- Connection definitions from Airflow are not available, so an ECS Task needs
  to be provided the necessary connection details (hostname, credentials).

While these pieces of functionality certainly can be put to good use, they have
not presented unsurmountable challenges to us.  We were able to:
- Join sensor tasks into the preceding tasks if that launched an external job
  (e.g. Glue, Sagemaker), or
- create common operators in the remaining skeleton `airflow-dags` project for
  those tasks (e.g. sensors) that were common to many DAGs.
- Drop `xcom` use.  It was not used a lot, we had no problem eliminating it.
- Specify connection details with each DAG that needs them.  We see this as
  a positive change.

## Conclusion
By moving our DAG definitions to the projects that need them, we were able
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
  - all architectural components are managed by our project

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
