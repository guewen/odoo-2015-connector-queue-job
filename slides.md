layout: false
class: center, middle, inverse

# A Jobs Queue for processing tasks asynchronously

---
# About us

.half-left[
## Guewen Baconnier
* Developer @ Camptocamp
* OCA committer, OCA Delegate
* Connector Team leader
* @guewenb
]

.half-right[
## Leonardo Pistone
* Developer @ Camptocamp
* OCA committer, OCA Delegate
* @lepistone
]

.big-camptocamp-logo[![Camptocamp](camptocamp.png)]

---
class: center, middle
# Computers are slow!
--

# Humans want them to be fast!


---
class: center, middle, inverse

# The problem

---
name: why-sequence
background-image: url(sequence_jobs.svg)

???
Story:
User clicks on a button.
Request goes to the server.
The server works. A while.
The user waits until the server responds.

So the waiting looks like...

---
class: center, middle

![Loading...](loading1.png)

# Loading...

---
class: center, middle

![Still loading...](loading2.png)

# Still loading...

---
class: center, middle

![Still loading... Please be patient.](loading3.png)

# Still loading... Please be patient.

---
class: center, middle

![Don't leave yet, it's still loading](loading4.png)

# Don't leave yet, it's still loading

---
class: center, middle

![You may not believe it, but the application is actually loading...](loading5.png)

# You may not believe it, but the application is actually loading...

---
class: center, middle

![Take a minute to get a coffee, because it's loading...](loading6.png)

# Take a minute to get a coffee, because it's loading...

---
class: center, middle

![Maybe you should consider reloading the application by pressing F5...](loading7.png)

# Come on...

---
class: center, middle, inverse

# If we can't make computers faster, we still have solutions

---
name: why-sequence2
background-image: url(sequence_jobs2.svg)

???

Illustration legend:
Now the requests of the user postpone an asynchronous job and directly returns a reponse.
The user is able to work on another thing.
A Connector Runner will execute the job as soon as possible.

---
class: center, middle, inverse
.connector-word-title[Connector]

???

How do we do asynchronous jobs in Odoo then?

Connector is an OCA addons.

Used in many addons.
Magento. Prestashop. Various connectors.

But also some which are not connectors, addons that use only the jobs queue.
Example: module (base_import_async) to import large CSV files in jobs, chunk by
chunk.

Tagline:
Framework to build connectors.
But not limited to connectors.
It's a Jobs Queue!


---
# Queue it!

Dependency on .connector-word[connector]

--

Declare a job:

```python
from openerp.addons.connector.queue.job import job

@job
def a_heavy_task(session, model_name, record_id):
    # do an heavy task on record_id of model_name
```

???

Decorate a function with @job.
The function is still callable synchronously, but also asynchrously if it is called with '.delay()'.

--
Delay a job:

```python
session = ConnectorSession.from_env(self.env)
a_heavy_task.delay(session, 'res.partner', 1)
```


???

Have to build a ConnectorSession with the Odoo env.
Then call the function with .delay().
That's it. The job is pushed in the queue and the execution continues to the next line.

---

# Dequeue it!

Start the server with:

```python
ODOO_CONNECTOR_CHANNELS=root:4 ./openerp-server --load=web,connector
```

???

Where workers > 1 and > the number next to root, so > 4.
But what are these channels?

---
class: inverse, center, middle

# Channels

???

Channels are related to how the jobs are executed.
Next slide shows illustrates a story.

---

background-image: url(channels_simple.svg)

???
Elements of the story:
 * jobs importing very large files
 * jobs making a lot of sync with magento or another shop
 * we want to execute 3 jobs at a time at max
 * we can't block the e-commerce sync during the import of the files
 * so the channel for importing very large files is limited to 1 job at a time

---
class: inverse, center, middle

# Properties

---

background-image: url(job_priority.svg)
# Priority

```python
a_task.delay(session, 1)  # default is 10
a_task.delay(session, 2, priority=50)
a_task.delay(session, 10, priority=999)
```

???
add a schema?

---

background-image: url(job_eta.svg)
# ETA

```python
a_task.delay(session, 1)  # A
a_task.delay(session, 1, eta=6*60*60)  # B
a_task.delay(session, 2, eta=datetime.now() + timedelta(days=1))  # C
```

???
add a schema?

---

# Retries

```python
a_task.delay(session, 1, max_retries=3)
```
--
# Invoke a retry


```python
@job
def a_task(session, args):
    try:
        do_operation()
    except (socket.gaierror, socket.error, socket.timeout) as err:                                                                                              
        raise RetryableError(                                                                                                                            
            'A network error caused the failure of the job: '                                                                                                   
            '%s' % err)
```

???

Usually, when an exception happens during the execution of a job, the job is set as failed. Failed jobs are shown on a view in Odoo with their traceback so they can be investigated.
Though for some exceptions, we know that the exception is temporary and the job could work later, such as a timeout.
We can catch such exceptions and instead raise a RetryError, which will put the job in the queue again some time later.

---
class: inverse, center, middle
# Best Practices
---

layout: false
.left-column[
  ## Outdating
]
.right-column[
Data in jobs can become outdated.

No:
```python
@job
def example(session, record_id, vals):
    export(record_id, vals)
```

Yes:
```python
 @job
 def example(session, record_id):
*    export(session.env['model'].browse(record_id))
```

]

???
The values of a record could change between the creation of a job and its execution.
Read the records again!

---

.left-column[
  ## Outdating
  ## Existence
]

.right-column[
A job can refer to a record which has been deleted.
Always check if it still exists.

No:
```python
@job
def example(session, record_id):
    export(session.env['model'].browse(record_id))
```

Yes:
```python
 @job
 def example(session, record_id):
     record = session.env['model'].browse(record_id)
*    if record.exists():
         export(record)
```

]

???

A job can refer to a deleted record, check if it exists.

---

.left-column[
  ## Outdating
  ## Existence
  ## Idempotence
]

.right-column[
A job should, when possible, produce the same result when executed several
times.

No:
```python
@job
def example(session, record_id):
    export(session.env['model'].browse(record_id))
```

Yes:
```python
 @job
 def example(session, record_id):
     record = session.env['model'].browse(record_id)
     if record.exists():
*        if not record.exported:
             export(session.env['model'].browse(record_id))
```

]

???

Always think what will happen if the same job is executed 2 times, it should not break anything.

---
class: inverse, center, middle

# Useful Patterns

---
# Fanout Job

A job generating other jobs.

```python
@job
def import_file(session, filepath):
    with open(filepath) as f:
        for line in f:
            import_line.delay(session, line)
```

???

Very often used when we have a batch of records to import or export.
For instance, we get all the lines of a file and import them separately.
Or we get all the modified ids in a period and we delay a job for each id.
Typically jobs with a high priority so there are executed before the imports of the records themselves.

---
# Try or delay

If an operation failed, try it later.

```python
@job
def do_operation(session, args):
    # work

try:
    do_operation(session, args)
except TimeoutError:
    do_operation.delay(session, args, eta=10*60)

```
???

The user clicks on a button and the synchronous operation failed, instead of
returning the error to the user, we postpone the task later.

---
# Extract highly concurrent tasks

And put them in a one-by-one channel.



???

Story: a lot of users create account moves which are configured to be validated
automatically. Each transaction tries to get a sequence and users get concurrent transaction errors.

Solution: extract the validation part in an asychronous job, so users create the move and continue to work. The jobs are executed one by one in a channel so we never have concurrent accesses.

---
class: center
background-image: url(all_sponsors.png)

# OCA Booth

