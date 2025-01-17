---
title: "Cross-DAG Dependencies"
description: "How to implement dependencies between your Airflow DAGs."
date: 2021-06-07T00:00:00.000Z
slug: "cross-dag-dependencies"
tags: ["DAGs", "Subdags", "Dependencies", "Sensors"]
---

## Overview

When designing Airflow DAGs, it is often best practice to put all related tasks in the same DAG. However, it's sometimes necessary to create dependencies between your DAGs. In this scenario, one node of a DAG is its own complete DAG, rather than just a single task. Throughout this guide, we'll use the following terms to describe DAG dependencies:

- Upstream DAG: A DAG that must reach a specified state before a downstream DAG can run
- Downstream DAG: A DAG that cannot run until an upstream DAG reaches a specified state

According to the Airflow documentation on [cross-DAG dependencies](https://airflow.apache.org/docs/apache-airflow/stable/howto/operator/external_task_sensor.html#cross-dag-dependencies), designing DAGs in this way can be useful when:

- Two DAGs are dependent, but they have different schedules.
- Two DAGs are dependent, but they are owned by different teams.
- A task depends on another task but for a different execution date.

For any scenario where you have dependent DAGs, we've got you covered! In this guide, we'll discuss multiple methods for implementing cross-DAG dependencies, including how to implement dependencies if your dependent DAGs are located in different Airflow deployments.

> Note: All code in this guide can be found in [this Github repo](https://github.com/astronomer/cross-dag-dependencies-tutorial).

## Implementing Cross-DAG Dependencies

There are multiple ways to implement cross-DAG dependencies in Airflow, including the `TriggerDagRunOperator`, `ExternalTaskSensor`, and the Airflow API. You'll want to use different implementations depending on your DAG dependencies and Airflow setup. In this section, we detail how to use each method and ideal scenarios for each, as well as how to view dependencies in the Airflow UI.

> **Note:** It can be tempting to use SubDAGs to handle DAG dependencies, but we highly recommend against doing so as SubDAGs can create performance issues. Instead, use one of the other methods described below.


### TriggerDagRunOperator

The `TriggerDagRunOperator` is an easy way to implement cross-DAG dependencies. This operator allows you to have a task in one DAG that triggers another DAG in the same Airflow environment. Read more in-depth documentation about this operator on the [Astronomer Registry](https://registry.astronomer.io/providers/apache-airflow/modules/triggerdagrunoperator).

The `TriggerDagRunOperator` is ideal in situations where you have one upstream DAG that needs to trigger one or more downstream DAGs, or if you have dependent DAGs that have both upstream and downstream tasks in the upstream DAG (i.e. the dependent DAG is in the middle of tasks in the upstream DAG). Because you can use this operator for any task in your DAG, it is highly flexible. It's also an ideal replacement for SubDAGs. 

Below is an example DAG that implements the `TriggerDagRunOperator` to trigger the `dependent-dag` between two other tasks. 

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.operators.trigger_dagrun import TriggerDagRunOperator
from datetime import datetime, timedelta

def print_task_type(**kwargs):
    """
    Dummy function to call before and after dependent DAG.
    """
    print(f"The {kwargs['task_type']} task has completed.")

# Default settings applied to all tasks
default_args = {
    'owner': 'airflow',
    'depends_on_past': False,
    'email_on_failure': False,
    'email_on_retry': False,
    'retries': 1,
    'retry_delay': timedelta(minutes=5)
}

with DAG('trigger-dagrun-dag',
         start_date=datetime(2021, 1, 1),
         max_active_runs=1,
         schedule_interval='@daily',
         default_args=default_args,
         catchup=False
         ) as dag:

    start_task = PythonOperator(
        task_id='starting_task',
        python_callable=print_task_type,
        op_kwargs={'task_type': 'starting'}
    )

    trigger_dependent_dag = TriggerDagRunOperator(
        task_id="trigger_dependent_dag",
        trigger_dag_id="dependent-dag",
        wait_for_completion=True
    )

    end_task = PythonOperator(
        task_id='end_task',
        python_callable=print_task_type,
        op_kwargs={'task_type': 'ending'}
    )

    start_task >> trigger_dependent_dag >> end_task
```

In the following graph view, you can see that the `trigger_dependent_dag` task in the middle is the `TriggerDagRunOperator`, which runs the `dependent-dag`. 

![Trigger DAG Graph](https://assets2.astronomer.io/main/guides/cross-dag-dependencies/trigger_dag_run_graph.png)

There are a couple of things to note when using this operator:

- If your dependent DAG requires a config input or a specific execution date, these can be specified in the operator using the `conf` and `execution_date` params respectively.
- If your upstream DAG has downstream tasks that require the downstream DAG to finish first, you should set the `wait_for_completion` param to `True` as shown in the example above. This param defaults to `False`, meaning once the downstream DAG has started, the upstream DAG will mark the task as a success and move on to any downstream tasks.

### ExternalTaskSensor

The next method for creating cross-DAG dependencies is to add an `ExternalTaskSensor` to your downstream DAG. The downstream DAG will wait until a task is completed in the upstream DAG before moving on to the rest of the DAG. You can find more info on this sensor on the [Astronomer Registry](https://registry.astronomer.io/providers/apache-airflow/modules/externaltasksensor).

This method is not as flexible as the `TriggerDagRunOperator`, since the dependency is implemented in the downstream DAG. It is ideal in situations where you have a downstream DAG that is dependent on multiple upstream DAGs. An example DAG using the `ExternalTaskSensor` is shown below:

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.sensors.external_task import ExternalTaskSensor
from datetime import datetime, timedelta

def downstream_fuction():
    """
    Downstream function with print statement.
    """
    print('Upstream DAG has completed. Starting other tasks.')

default_args = {
    'owner': 'airflow',
    'depends_on_past': False,
    'email_on_failure': False,
    'email_on_retry': False,
    'retries': 1,
    'retry_delay': timedelta(minutes=5)
}

with DAG('external-task-sensor-dag',
         start_date=datetime(2021, 1, 1),
         max_active_runs=3,
         schedule_interval='*/1 * * * *',
         catchup=False
         ) as dag:

    downstream_task1 = ExternalTaskSensor(
        task_id="downstream_task1",
        external_dag_id='example_dag',
        external_task_id='bash_print_date2',
        allowed_states=['success'],
        failed_states=['failed', 'skipped']
    )

    downstream_task2 = PythonOperator(
        task_id='downstream_task2',
        python_callable=downstream_fuction,
        provide_context=True
    )

    downstream_task1 >> downstream_task2
```

In this DAG, `downstream_task1` waits for the `bash_print_date2` task of `example_dag` to complete before moving on to execute the rest of the downstream tasks (`downstream_task2`). The graph view of the DAG looks like this:

![External Task Sensor Graph](https://assets2.astronomer.io/main/guides/cross-dag-dependencies/external_task_sensor_graph.png)

If you want the downstream DAG to wait for the entire upstream DAG to finish instead of a specific task, you can set the `external_task_id` to `None`. In this case, we specify that the external task must have a state of `success` for the downstream task to succeed, as defined by the `allowed_states` and `failed_states`. 

Also note that in the example above, the upstream DAG (`example_dag`) and downstream DAG (`external-task-sensor-dag`) must have the same start date and schedule interval. This is because the `ExternalTaskSensor` will look for completion of the specified task or DAG at the same `execution_date`. To look for completion of the external task at a different date, you can make use of either of the `execution_delta` or `execution_date_fn` parameters (these are described in more detail in the documentation linked above).

### Airflow API

The [Airflow API](https://airflow.apache.org/docs/apache-airflow/stable/stable-rest-api-ref.html) is another way of creating cross-DAG dependencies. This is especially useful in [Airflow 2.0](https://www.astronomer.io/blog/introducing-airflow-2-0), which has a full stable REST API. To use the API to trigger a DAG run, you can make a POST request to the `DAGRuns` endpoint as described in the [Airflow documentation](https://www.astronomer.io/blog/introducing-airflow-2-0).

This method is useful if your dependent DAGs live in different Airflow environments (more on this in the Cross-Deployment Dependencies section below), or if the downstream DAG does not have any downstream dependencies in the upstream DAG (e.g. the downstream DAG is the last task in the upstream DAG). One drawback to this method is that the task triggering the downstream DAG will complete once the API call is complete, rather than when the downstream DAG is complete.

Using the API to trigger a downstream DAG can be implemented within a DAG by using the `SimpleHttpOperator` as shown in the example DAG below:

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.http.operators.http import SimpleHttpOperator
from datetime import datetime, timedelta
import json

# Define body of POST request for the API call to trigger another DAG
date = '{{ execution_date }}'
request_body = {
  "execution_date": date
}
json_body = json.dumps(request_body)

def print_task_type(**kwargs):
    """
    Dummy function to call before and after downstream DAG.
    """
    print(f"The {kwargs['task_type']} task has completed.")
    print(request_body)

default_args = {
    'owner': 'airflow',
    'depends_on_past': False,
    'email_on_failure': False,
    'email_on_retry': False,
    'retries': 1,
    'retry_delay': timedelta(minutes=5)
}

with DAG('api-dag',
         start_date=datetime(2021, 1, 1),
         max_active_runs=1,
         schedule_interval='@daily',
         catchup=False
         ) as dag:

    start_task = PythonOperator(
        task_id='starting_task',
        python_callable=print_task_type,
        op_kwargs={'task_type': 'starting'}
    )

    api_trigger_dependent_dag = SimpleHttpOperator(
        task_id="api_trigger_dependent_dag",
        http_conn_id='airflow-api',
        endpoint='/api/v1/dags/dependent-dag/dagRuns',
        method='POST',
        headers={'Content-Type': 'application/json'},
        data=json_body
    )

    end_task = PythonOperator(
        task_id='end_task',
        python_callable=print_task_type,
        op_kwargs={'task_type': 'ending'}
    )

    start_task >> api_trigger_dependent_dag >> end_task
```

This DAG has a similar structure to the `TriggerDagRunOperator` DAG above, but instead uses the `SimpleHttpOperator` to trigger the `dependent-dag` using the Airflow API. The graph view looks like this:

![API Graph View](https://assets2.astronomer.io/main/guides/cross-dag-dependencies/api_graph.png)

In order to use the `SimpleHttpOperator` to trigger another DAG, you need to define the following:

- `endpoint`: This should be of the form `'/api/v1/dags/<dag-id>/dagRuns'` where `<dag-id>` is the ID of the DAG you want to trigger.
- `data`: To trigger a DAG Run using this endpoint, you must provide an execution date. In the example above, we use the `execution_date` of the upstream DAG, but this can be any date of your choosing. You can also specify other information about the DAG run as described in the API documentation linked above.
- `http_conn_id`: This should be an Airflow connection of type HTTP, with your Airflow domain as the Host. Any authentication should be provided either as a Login/Password (if using Basic auth) or as a JSON-formatted Extra. In the example below, we use an authorization token.

![Http Connection](https://assets2.astronomer.io/main/guides/cross-dag-dependencies/http_connection.png)

### DAG Dependencies View

In [Airflow 2.1](https://airflow.apache.org/docs/apache-airflow/stable/changelog.html#airflow-2-1-0-2021-05-21), a new cross-DAG dependencies view was added to the Airflow UI. This view shows all dependencies between DAGs in your Airflow environment as long as they are implemented using one of the following methods:

- Using a `TriggerDagRunOperator`
- Using an `ExternalTaskSensor`

Dependencies can be viewed in the UI by going to `Browse` → `DAG Dependencies`. The screenshot below shows the dependencies created by the `TriggerDagRunOperator` and `ExternalTaskSensor` example DAGs in the sections above.

![DAG Dependencies View](https://assets2.astronomer.io/main/guides/cross-dag-dependencies/dag_dependencies_view.png)


## Cross-Deployment Dependencies

Sometimes it may be necessary to implement cross-DAG dependencies where the DAGs do not exist in the same Airflow deployment. The `TriggerDagRunOperator` and `ExternalTaskSensor` methods described above are designed to work with DAGs in the same Airflow environment, so they are not ideal for cross-Airflow deployments. The Airflow API, on the other hand, is perfect for this use case. In this section, we'll focus on how to implement this method on Astro, but the general concepts will likely be similar wherever your Airflow environments are deployed.

### Cross-Deployment Dependencies with Astronomer

To implement cross-DAG dependencies on two different Airflow environments on Astro, we can follow the same general steps for triggering a DAG using the Airflow API described above. It may be helpful to first read our documentation on [making requests to the Airflow API](https://docs.astronomer.io/software/airflow-api) from Astronomer. When you're ready to implement a cross-deployment dependency, follow these steps:

1. In the upstream DAG, create a `SimpleHttpOperator` task that will trigger the downstream DAG. Refer to the section above for details on configuring the operator.
2. In the downstream DAG Airflow environment, [create a Service Account](https://docs.astronomer.io/software/ci-cd#step-1-create-a-service-account) and copy the API key.
3. In the upstream DAG Airflow environment, create an Airflow connection as shown in the Airflow API section above. The Host should be `https://<your-base-domain>/<deployment-release-name>/airflow` where the base domain and deployment release name are from your downstream DAG's Airflow deployment. In the Extras, use `{"Authorization": "api-token"}` where `api-token` is the service account API key you copied in step 2.
4. Ensure the downstream DAG is turned on, then run the upstream DAG.
