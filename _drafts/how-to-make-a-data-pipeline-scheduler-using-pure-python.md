---
title: ""
layout: post
permalink: /drafts/how-to-make-a-data-pipeline-scheduler-using-pure-python.md
----

# How to make a data pipeline scheduler using pure Python

Building data pipelines is hard. Data engineers reach out for tools like Airflow and Luigi to help them building a solid data pipeline system. *But did you know you can build your own minimalistic scheduler using standard Python only?*

In this blogpost, we will build simple scheduler for your data pipelines step by step.

## Example case: webscraping
Assume you have a system to scrape some websites and extra phrases to feed a language model. 
The system is build out with three data pipelines:

1. A pipeline that generates a list of urls based a domain by crawling sitemaps.
2. A fetching pipeline that downloads a web page given a url.
3. An extraction pipeline that parses the raw  HTML web page and outputs a text file.

```python
import re
from pathlib import Path
import requests
import pandas as pd
from bs4 import BeautifulSoup

def sanitize_fname(fname):
    return re.sub('[^0-9a-zA-Z]+', '-', fname)

def collect_urls(domain: str,filepath: Path):
    """Pipeline to collect the urls of the blogposts"""
    sitemap_url = f"{domain}/sitemap.xml"
    response = requests.get(sitemap_url)
    def generate_urls(df_sitemap):
        for _, row in df_sitemap.iterrows():
            # check if the url is a sitemap
            if row["loc"].endswith("sitemap.xml"):
                # if so, fetch the sitemap and recurse
                response = requests.get(row["loc"])
                df_sitemap = pd.read_xml(response.text)
                yield from generate_urls(df_sitemap)
            yield row["loc"] 
    df_sitemap = pd.read_xml(response.text)
    # save the urls to a file
    with open(filepath, "w") as f:
        for url in generate_urls(df_sitemap):
            f.write(url + "\n")

def fetch_page(url:str, filepath:Path):
    """Pipeline to fetch a page from the web and return the content"""
    response = requests.get(url)
    with open(filepath, "wb") as f:
        f.write(response.content)

def parse_page(input_filepath:Path, output_filepath:Path):
    """Pipeline to parse the page and extract content"""
    with open(input_filepath, "r") as f:
        html = f.read()
    soup = BeautifulSoup(html, "html.parser")
    content = soup.find("div", {"class": "entry-content"}).text
    with open(output_filepath, "w") as f:
        f.write(content)

if __name__ == "__main__":
    data_path = Path("./data")

    for i,domain in enumerate(DOMAINS):
        for url in collect_urls(domain, data_path / f"urls_{}.txt")
            fetch_page(url, data_path / f"{sanitize_fname(url)}.html")
            parse_page(data_path / f"{sanitize_fname(url)}.html", data_path / f"{sanitize_fname(url)}.txt")

```

Some importent things to note here:
- Our pipelines are designed to be parallizeable. The URL-scraping pipeline can be runned in parallel per domain. The fetch and the extraction pipeline can be run per page. 
- Also, the pipelines are dependent on each other, which is important to keep in mind when executing them in parallel.

### Minor refactoring of the functions

Ok. Before we start building the scheduler. Let's refactor the functions a little bit. This will make them more suitable for multiprocessing

```python
### BEFORE
def parse_page(input_filepath:Path, output_filepath:Path):
    # body stays identical

## AFTER

@dataclass
class ParsePage
    input_filepath: Path
    output_filepath: Path

    def run():
        # body comes here
```

Using data classes we can easily seperate the creation and invocation of a task:
 ```python
 # creating the task is calling the constructor
 task = ParsePage("./raw_html/page1.html", "./parsed/page1.txt")
 # invoking the task is calling the run() method
 task.run()
 ```
> Note: a 'task' = 'data pipeline' + 'some parameters'

## The ingredients 

Our scheduler has three important ingredients: a depedenncy resolver, multiprocessing queue and a status map.

### Resolve dependencies

Our scheduler will have some form of a task queue where every task is a pipeline with some parameters. The scheduler can't just naïvely execute the tasks one by one becaus the HTML-parsers need be executed after the webpage fetchers.
Since our tasks are dependent on each other, the scheduler need to continuously evaluate the complete tasks and release the tasks for which all upstream dependencies have been completed. 

Luckily Python has a **built-in package** `graphlib` that implements an algorithm called Toplogical Sort. This algorithm efficiently sorts tasks in a way that all tasks will be executed while respecting their dependencies:

Simple example:
```python
from graphlib import TopologicalSorter

graph = {"D": {"B", "C"}, "C": {"A"}, "B": {"A"}}
ts = TopologicalSorter(graph)
sorted_nodes = tuple(ts.static_order())
print("Sorted Nodes: ")
print(sorted_nodes)
# ('A', 'C', 'B', 'D' )
```

If use this topological sorter on our webscraping system we end up with the following task queue generator:

```python
DOMAINS = ["http://uzleuven.be", "http://uzbrussel.be"]
def generate_dependencies():
    for domain in DOMAINS:


```

### Multiprocessing workers
One of the goals of our scheduler is that it should be able to **execute tasks on a cluster of worker in parallel**. The simplest way to achieve this in Python is using `multiprocessing`. But using multiprocessing comes at a cost. We're losing the shared memory space and can't call the `task.run()` method on the worker directly from the scheduler. We need to use queues for communication between the scheduler and the workers. In our case, a worker needs two communication channels: one to receive work (tasks) and one to tell we're done or there was a problem with a task.

In Python this comes down to a simple while-loop that keeps the working a live a the exact steps we've describe above: read task, execute task, report status:

```python
from multiprocessing import JoinableQueue

def worker(work_queue:JoinableQueue, status_queue:JoinableQueue):
    """Execute work from the work queue and reports success or failure on the status queue"""
    while True:
        taks = work_queue.get() # retrieve a task
        try:
            task.run() # run it
        except Exception:
            status_queue.put((task, "FAILED"))
        else:
            status_queue.put((task, "SUCCES"))

```

I hope this simple piece of code makes it clear why we refactored our data pipelines from functions into classes. The workers don't need to know any detail about the configuration of the task. They only need to know the `run()` method.

### Putting together the scheduler

The scheduler has

```python
from multiprocessing import JoinableQueue, Manager

def worker(task_queue:JoinableQueue, status_queue:JoinableQueue):
    ...

def schedule(graph, /, work_queue:JoinableQueue, status_queue:JoinableQueue):
    """Schedules a sequence of task graphs to parallel works while respecting the depencies in the graph"""
    status_map = {}
    task_topsort_map = {}
    failed_graphs = []

    ts = TopologicalSorter(graph)
    ts.prepare()

    active_ts = [ts]
    while len(active_ts) > 0:
        # iterate through all top. sorters. to schedule 

        # 1. schedule all tasks that are ready 
        for ts in active_ts:
            if active_ts.is_active():
                for task in active_ts.get_ready()
                    if task in status_map and status_map[task] == "ERROR":
                        continue # TODO fix failed graphs
                    task_topsort_map[task] = ts
                    task_queue.put(task)
            else:
                # the graph is ready 🎉
                active_ts.remove(graph)

        # 2. wait for next task to be finalized
        (finalized_task, status) = status_queue.get()
        # update status map
        status_map[finalized_task] = status
        
        # mark task as done so and release downstream tasks
        if status == "SUCCESS" # ✅
            task_topsort_map[task].done(task) # release tasks in TS
            # check if we need to add new downstream tasks
            downstream_tasks = finalized_task.downtream_tasks() 
            ts = create_task_sorter(*downstream_tasks)
            active_ts.append(ts)
        else:
            # handle error


NUM_PROCESSES = 4

if __name__ == "__main__" : 
    manager = Manager()
    task_queue = JoinableQueue()
    finalized_tasks_queue = JoinableQueue()

    for _ in range(NUM_PROCESSES):
        process = multiprocessing.Process(target=worker,
                                            args=(task_queue, finalized_tasks_queue),
                                            daemon=True)
        processes.append(process)
        process.start()
```

__Important remarks__
-  We assume that the duration of a single cycle of the scheduler is magnitudes faster than processing time of one task
-  Calculating `downstream_tasks` can be expensive and consequently blocking which may break our first assumptions
-  Generating downstreak tasks dynamically instead of a constructing a large task graph upfront is powerfull but comes with a big responsibility. Cycles and infintie loops can't be detected easily. This is something a TopologicalSorter *can* detect beforehand.

- Decoupling our functions and using a graph like approach to link data pipelines together is a significant tax if you look at the lines of code. Instead of imperatively passing the returned value from one function as the argument of next function results we built a declarative syntax that allows us to build the task graph upfront. The positive side is that we can serialize the whole graph, inspect it for cycles and execute it in parallel efficiently.