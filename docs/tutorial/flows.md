---
description: Learn the basics of defining and running flows.
tags:
    - tutorial
    - getting started
    - basics
    - flows
    - logging
    - parameters
    - retries
---
!!! note "Start a Prefect API and UI"
    This tutorial is best paired with a Prefect UI so that you can see the information that Prefect is capturing.  
    If using Prefect Cloud, navigate to your workspace at [`https://app.prefect.cloud/`](https://app.prefect.cloud/).
    If using a self-hosted setup, run `prefect server start` to run both the webserver and UI. 

## What is a flow?

[Flows](/concepts/flows/) are like functions. They can take inputs, perform work, and return an output. In fact, you can turn any function into a Prefect flow by adding the `@flow` decorator. When a function becomes a flow, its behavior changes, giving it the following advantages:

- All runs of the flow have persistent [state](/concepts/states/). Transitions between states are recorded, allowing for flow execution to be observed and acted upon.
- Input arguments can be type validated as workflow parameters.
- Retries can be performed on failure.
- Timeouts can be enforced to prevent unintentional, long-running workflows.
- Metadata about [flow runs](#flow-runs), such as run time and final state, is automatically tracked.
- They can easily be elevated to a [deployment](/concepts/deployments/), which exposes a remote API for interacting with it

## Run your first flow

The simplest way to get started with Prefect is to annotate a Python function with the `@flow` decorator. 
The script below fetches statistics about the [main Prefect repository](https://github.com/PrefectHQ/prefect). 
Let's turn it into a Prefect flow and run it:

```python title="repo_info.py" hl_lines="2 5"
import httpx
from prefect import flow


@flow
def get_repo_info():
    url = "https://api.github.com/repos/PrefectHQ/prefect"
    response = httpx.get(url)
    response.raise_for_status()
    repo = response.json()
    print("PrefectHQ/prefect repository statistics 🤓:")
    print(f"Stars 🌠 : {repo['stargazers_count']}")
    print(f"Forks 🍴 : {repo['forks_count']}")
```

Running this file interactively (using `python -i`) and calling this function directly will result in some interesting output:

<div class="terminal">
```bash
python -i repo_info.py
>>> get_repo_info()

12:47:42.792 | INFO | prefect.engine - Created flow run 'ludicrous-warthog' for flow 'get-repo-info'
PrefectHQ/prefect repository statistics 🤓:
Stars 🌠 : 12146
Forks 🍴 : 1245
12:47:45.008 | INFO | Flow run 'ludicrous-warthog' - Finished in state Completed()
```
</div>

!!! tip "Flows can contain arbitrary Python"
    As we can see above, flow definitions can contain arbitrary Python logic.

## Parameters

As with any Python function, you can pass arguments to a flow. 
The positional and keyword arguments defined on your flow function are called [parameters](/concepts/flows/#parameters). 
Prefect will automatically perform type conversion using any provided type hints. 
Let's make the repository a string parameter with a default value:

```python hl_lines="6 7 11" title="repo_info.py"
import httpx
from prefect import flow


@flow
def get_repo_info(repo_name: str = "PrefectHQ/prefect"):
    url = f"https://api.github.com/repos/{repo_name}"
    response = httpx.get(url)
    response.raise_for_status()
    repo = response.json()
    print(f"{repo_name} repository statistics 🤓:")
    print(f"Stars 🌠 : {repo['stargazers_count']}")
    print(f"Forks 🍴 : {repo['forks_count']}")
```

Running `python -i repo_info.py` we can now call our flow with varying values for the `repo_name` parameter (including "bad" values):

<div class="terminal">
```bash
python -i repo_info.py
>>> get_repo_info(repo_name="PrefectHQ/marvin")
...
>>> get_repo_info(repo_name="missing-org/missing-repo")
HTTPStatusError: Client error '404 Not Found' for url 'https://api.github.com/repos/missing-org/missing-repo'
```
</div>

Now navigate to your Prefect dashboard and compare the displays for these two runs.

## Logging

Prefect enables you to log a variety of useful information about your flow and task runs, capturing information about your workflows for purposes such as monitoring, troubleshooting, and auditing. 
If we navigate to our dashboard and explore the runs we created above, we will notice that the repository statistics are not captured in the flow run logs.  
Let's fix that by adding some [logging](/concepts/logs) to our flow:

```python hl_lines="2 11-14" title="repo_info.py"
import httpx
from prefect import flow, get_run_logger


@flow
def get_repo_info(repo_name: str = "PrefectHQ/prefect"):
    url = f"https://api.github.com/repos/{repo_name}"
    response = httpx.get(url)
    response.raise_for_status()
    repo = response.json()
    logger = get_run_logger()
    logger.info("%s repository statistics 🤓:", repo_name)
    logger.info(f"Stars 🌠 : %d", repo["stargazers_count"])
    logger.info(f"Forks 🍴 : %d", repo["forks_count"])
```

Now the output looks more consistent _and_, more importantly, our statistics are stored in the Prefect backend and displayed in the UI for this flow run:

<div class="terminal">
```bash
12:47:42.792 | INFO    | prefect.engine - Created flow run 'ludicrous-warthog' for flow 'get-repo-info'
12:47:43.016 | INFO    | Flow run 'ludicrous-warthog' - PrefectHQ/prefect repository statistics 🤓:
12:47:43.016 | INFO    | Flow run 'ludicrous-warthog' - Stars 🌠 : 12146
12:47:43.042 | INFO    | Flow run 'ludicrous-warthog' - Forks 🍴 : 1245
12:47:45.008 | INFO    | Flow run 'ludicrous-warthog' - Finished in state Completed()
```
</div>

!!! tip "`log_prints=True`"
    We could have achieved the exact same outcome by using Prefect's convenient `log_prints` keyword argument in the `flow` decorator:
    ```python
    @flow(log_prints=True)
    def get_repo_info(repo_name: str = "PrefectHQ/prefect"):
        ...
    ```

!!! warning "Logging vs Artifacts"
    The example above is for educational purposes.
    In general, it is better to use [Prefect artifacts](/concepts/artifacts/) for storing metrics and output.
    Logs are best for tracking progress and debugging errors.

## Retries

So far our script works, but in the future unexpected errors may occur; for example the GitHub API may be temporarily unavailable or rate limited. 
[Retries](/concepts/flows/#flow-settings) help make our flow more resilient. 
Let's add retry functionality to our example above:
```python hl_lines="5" title="repo_info.py"
import httpx
from prefect import flow


@flow(retries=3, retry_delay_seconds=5, log_prints=True)
def get_repo_info(repo_name: str = "PrefectHQ/prefect"):
    url = f"https://api.github.com/repos/{repo_name}"
    response = httpx.get(url)
    response.raise_for_status()
    repo = response.json()
    print(f"{repo_name} repository statistics 🤓:")
    print(f"Stars 🌠 : {repo['stargazers_count']}")
    print(f"Forks 🍴 : {repo['forks_count']}")
```

## [Next: Tasks](/tutorial/tasks/)

As you have seen, adding a flow decorator converts our Python function to a resilient and observable workflow. 
In the next section, you'll supercharge this flow by using tasks to break down the workflow's complexity and make it more performant and observable - [click here to continue](/tutorial/tasks/).
