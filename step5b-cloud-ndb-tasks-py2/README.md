# Step 5b - Migrate from `ndb` &amp; `taskqueue` to Cloud NDB &amp; Cloud Tasks

## Introduction

The goal of the Step 5 series of codelabs and repos like this is to help App Engine developers migrate from [Python 2 App Engine (Push) Task Queues](https://cloud.google.com/appengine/docs/standard/python/taskqueue/push) to [Google Cloud Tasks](https://cloud.google.com/tasks). They are meant to be *complementary* to the official [migrating push queues to Cloud Tasks documentation](https://cloud.google.com/appengine/docs/standard/python/taskqueue/push/migrating-push-queues) and [corresponding code samples](https://github.com/GoogleCloudPlatform/python-docs-samples/tree/master/appengine/standard/migration/taskqueue) and offer some additional benefits:
- Video content for those who prefer visual learning in addition to reading
- Codelab tutorials give hands-on experience and build "migration muscle-memory"
- More code samples gives developers a deeper understanding of migration steps

In short, these are the Step 5 codelabs/repos:
- Step 5a ([codelab](https://codelabs.developers.google.com/codelabs/cloud-gae-python-migrate-5a-gaetasksndb.md), [repo](/step5a-gae-ndb-tasks-py2)): Add push tasks (App Engine `taskqueue`) to Step 1 ([codelab](https://codelabs.developers.google.com/codelabs/cloud-gae-python-migrate-1-flask), [repo](/step1-flask-gaendb-py2)) Flask &amp; `ndb` Python 2 app
- Step 5b ([codelab](https://codelabs.developers.google.com/codelabs/cloud-gae-python-migrate-5b-cloudtasksndb.md), [repo](/step5b-cloud-ndb-tasks-py2)): Migrate from App Engine `ndb` &amp; `taskqueue` to Cloud NDB &amp; Cloud Tasks
- Step 5c ([codelab](https://codelabs.developers.google.com/codelabs/cloud-gae-python-migrate-5c-cloudtasksds.md), [repo](/step5c-cloud-ndb-tasks-py3)): Migrate Step 5b app to second-generation Python 3 App Engine &amp; Cloud Datastore

In *this* codelab/repo, participants start with the code in the (completed) [Step 5a repo](https://github.com/googlecodelabs/migrate-python-appengine-datastore/tree/master/step5a-gae-ndb-tasks-py2). That repo resulted from taking the [Step 1 sample app](https://github.com/googlecodelabs/migrate-python-appengine-datastore/tree/master/step1-flask-gaendb-py2) and adding push tasks to it using the App Engine `taskqueue` API library. This tutorial performs a pair of migrations from *that* starting point:
- Migrate from App Engine `ndb` to Cloud NDB (same as Step 2 [[codelab](https://codelabs.developers.google.com/codelabs/cloud-gae-python-migrate-2-cloudndb), [repo](https://github.com/googlecodelabs/migrate-python-appengine-datastore/tree/master/step2-flask-cloudndb-py2)])
- Migrate from App Engine `taskqueue` to Cloud Tasks

If you haven't completed the [Step 5a codelab](https://codelabs.developers.google.com/codelabs/cloud-gae-python-migrate-5-gaetasks), we recommend you do so to familiarize yourself with its codebase as we start from there. (You can also just study the code in its repo linked above.)

---

## Migration

The migration from App Engine `ndb` to Cloud NDB is identical to that of Step 2 already described above. That migration is *not* covered in this Step 5b codelab... it'll "just happen" in the code at the same time as the Tasks migration. If you need deeper coverage of the NDB migration, complete the Step 2 codelab linked above. The purpose is to have a similar starting point for Python 2 App Engine users with `ndb` &amp; push tasks apps. Here are the primary migration steps:

1. Update `requirements.txt` to include Google Cloud client libraries
1. Update `app.yaml` to reference built-in libraries required by the Cloud client libraries
1. Update `appengine_config.py` so App Engine can access those [built-in libraries](https://cloud.google.com/appengine/docs/standard/python/tools/built-in-libraries-27)
1. Switch application code to use the Cloud client libraries

### Configuration

The `requirements.txt` from Step 5a listed only Flask as a required package. Cloud NDB and Cloud Tasks have their own client libraries, so in this step, add their packages to `requirements.txt` so it looks like this (but choose the most recent versions as these were the latest at the time of this writing):

    Flask==1.1.2
    google-cloud-ndb==1.7.1
    google-cloud-tasks==1.5.0

Reference the `grpcio` &amp; `setuptools` built-in libraries in `app.yaml` in a `libraries` section:

```yml
libraries:
- name: grpcio
  version: 1.0.0
- name: setuptools
  version: 36.6.0
```

Update `appengine_config.py` to use `pkg_resources` to tie those built-in libraries to the bundled/vendored third-party libraries like Flask and the Google Cloud client libraries:

```python
import pkg_resources
from google.appengine.ext import vendor

# Set PATH to your libraries folder.
PATH = 'lib'
# Add libraries installed in the PATH folder.
vendor.add(PATH)
# Add libraries to pkg_resources working set to find the distribution.
pkg_resources.working_set.add_entry(PATH)
```

### Migrate from App Engine `ndb` &amp; `taskqueue` to Cloud NDB &amp; Cloud Tasks

#### Imports

Our app is currently using the built-in `google.appengine.api.taskqueue` &amp; `google.appengine.ext.ndb` libraries:

- BEFORE:

```python
from datetime import datetime
import logging
import time
from flask import Flask, render_template, request
from google.appengine.api import taskqueue
from google.appengine.ext import ndb
```

Replace both with `google.cloud.ndb` and `google.cloud.tasks`. Furthermore, Cloud Tasks requires you to JSON-encode the task's payload, so also import `json`. When you're done, here's what the `import` section of `main.py` should look like:


- AFTER:

```python
from datetime import datetime
import json
import logging
import time
from flask import Flask, render_template, request
from google.cloud import ndb, tasks
```

#### Migrate to Cloud Tasks (and Cloud NDB)

- BEFORE:

```python
def store_visit(remote_addr, user_agent):
    'create new Visit entity in Datastore'
    Visit(visitor='{}: {}'.format(remote_addr, user_agent)).put()
```

There is no change to `store_visit()` other than what you did in Step 2: add a context manager to all Datastore access. This comes in the form of moving creation of a new `Visit` Entity wrapped in a `with` statement.

- AFTER:

```python
def store_visit(remote_addr, user_agent):
    'create new Visit entity in Datastore'
    with ds_client.context():
        Visit(visitor='{}: {}'.format(remote_addr, user_agent)).put()
```

Cloud Tasks currently requires an App Engine be enabled for your Google Cloud project in order for you to use it (even if you don't have any App Engine code), otherwise tasks queues will not function. (See [this section](https://cloud.google.com/tasks/docs/creating-queues#before_you_begin) in the docs for more information.) Cloud Tasks supports tasks running on App Engine (App Engine "targets") but can also be run on any HTTP endpoint (HTTP targets) with a public IP address, such as Cloud Functions, Cloud Run, GKE, Compute Engine, or even an on-prem web server. Our simple app uses an App Engine target for tasks.

Some setup is needed to use Cloud NDB and Cloud Tasks. At the top of `main.py` under Flask initialization, initialize Cloud NDB and Cloud Tasks. Also define some constants that indicate where your push tasks will execute.

```python
app = Flask(__name__)
ds_client = ndb.Client()
ts_client = tasks.CloudTasksClient()

PROJECT_ID = 'PROJECT_ID'  # replace w/your own
REGION_ID = 'REGION_ID'    # replace w/your own
QUEUE_NAME = 'default'     # replace w/your own
QUEUE_PATH = ts_client.queue_path(PROJECT_ID, REGION_ID, QUEUE_NAME)
```
Obviously once you've [created your task queue](https://cloud.google.com/tasks/docs/creating-queues), fill-in your project's `PROJECT_ID`, the `REGION_ID` where your tasks will run (should be the same as your App Engine region), and the name of your push queue. App Engine features a "`default`" queue, so we'll use that name (but you don't have to).

The `default` queue is special and created automatically under certain circumstances, one of which is when using *App Engine APIs*, so if you (re)use the same project as Step 5a, `default` will already exist. However if you created a *new* project specifically for Step 5b, you'll need to create `default` manually. More info on the `default` queue can be found on [this page](https://cloud.google.com/tasks/docs/queue-yaml#cloud_tasks_and_the_default_app_engine_queue).

The purpose of `ts_client.queue_path()` is to create a task queue's "fully-qualified path name" (`QUEUE_PATH`) needed for creating a task. Also needed is a JSON structure specifying task parameters:

```python
task = {
    'app_engine_http_request': {
        'relative_uri': '/trim',
        'body': json.dumps({'oldest': oldest}).encode(),
        'headers': {
            'Content-Type': 'application/json',
        },
    }
}
```

What are you looking at above?
1. Supply task target information:
    - For App Engine targets, specify `app_engine_http_request` as the request type and `relative_uri` is the App Engine task handler.
    - For HTTP targets, use `http_request` &amp; `url` instead.
1. `body`: the JSON- and Unicode string-encoded parameter(s) to send to the (push) task
1. Specify a JSON-encoded `Content-Type` header explicity

Refer to the [documentation](https://cloud.google.com/tasks/docs/reference/rpc/google.cloud.tasks.v2#task) for more info on your options here.

With setup out of the way, let's update `fetch_visits()`. Here is what it looks like from the previous tutorial:

- BEFORE:

```python
def fetch_visits(limit):
    'get most recent visits & add task to delete older visits'
    data = Visit.query().order(-Visit.timestamp).fetch(limit)
    oldest = time.mktime(data[-1].timestamp.timetuple())
    oldest_str = time.ctime(oldest)
    logging.info('Delete entities older than %s' % oldest_str)
    taskqueue.add(url='/trim', params={'oldest': oldest})
    return (v.to_dict() for v in data), oldest_str
```

The required updates:
1. Switch from App Engine `ndb` to Cloud NDB
1. New code to extract timestamp of oldest visit displayed
1. Use Cloud Tasks to create a new task instead of App Engine `taskqueue`

Here's what your new `fetch_visits()` should look like:

- AFTER:

```python
def fetch_visits(limit):
    'get most recent visits & add task to delete older visits'
    with ds_client.context():
        data = Visit.query().order(-Visit.timestamp).fetch(limit)
    oldest = time.mktime(data[-1].timestamp.timetuple())
    oldest_str = time.ctime(oldest)
    logging.info('Delete entities older than %s' % oldest_str)
    task = {
        'app_engine_http_request': {
            'relative_uri': '/trim',
            'body': json.dumps({'oldest': oldest}).encode(),
            'headers': {
                'Content-Type': 'application/json',
            },
        }
    }
    ts_client.create_task(parent=QUEUE_PATH, task=task)
    return (v.to_dict() for v in data), oldest_str
```

Summarizing the code update:
- Switch to Cloud NDB means moving Datastore code inside a `with` statement
- Switch to Cloud Tasks means using `ts_client.create_task()` instead of `taskqueue.add()`
- Pass in the queue's full path and `task` payload (described earlier)


#### Update (Push) Task handler

There are very few changes that need to be made to the (push) task handler function.

- BEFORE:

```python
@app.route('/trim', methods=['POST'])
def trim():
    '(push) task queue handler to delete oldest visits'
    oldest = request.form.get('oldest', type=float)
    keys = Visit.query(
            Visit.timestamp < datetime.fromtimestamp(oldest)
    ).fetch(keys_only=True)
    nkeys = len(keys)
    if nkeys:
        logging.info('Deleting %d entities: %s' % (
                nkeys, ', '.join(str(k.id()) for k in keys)))
        ndb.delete_multi(keys)
    else:
        logging.info('No entities older than: %s' % time.ctime(oldest))
    return ''   # need to return SOME string w/200
```

The only thing that needs to be done, is to place all Datastore access within the context manager `with` statement, both the query and delete request. With this in mind, update your `trim()` handler like this:

- AFTER:

```python
@app.route('/trim', methods=['POST'])
def trim():
    '(push) task queue handler to delete oldest visits'
    oldest = float(request.get_json().get('oldest'))
    with ds_client.context():
        keys = Visit.query(
                Visit.timestamp < datetime.fromtimestamp(oldest)
        ).fetch(keys_only=True)
        nkeys = len(keys)
        if nkeys:
            logging.info('Deleting %d entities: %s' % (
                    nkeys, ', '.join(str(k.id()) for k in keys)))
            ndb.delete_multi(keys)
        else:
            logging.info('No entities older than: %s' % time.ctime(oldest))
    return ''   # need to return SOME string w/200
```


#### Web template

There are no changes to `templates/index.html` in this step nor Step 5c.

## Next

Deploy to App Engine and confirm everything still works. Once you're satisfied, move onto the next step:

- [**Step 5c:**](/step5c-cloud-datastore-tasks-py3) Migrate your Step 5b Cloud Tasks app from Python 2 to 3 and from Cloud NDB to Cloud Datastore

