# Celery-Redis-Flask Task Management Tutorial

## Table of Contents
- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
- [Setting Up the Environment](#setting-up-the-environment)
- [Creating a Simple Flask-Celery Application](#creating-a-simple-flask-celery-application)
- [Running Celery Worker](#running-celery-worker)
- [Submitting and Monitoring Tasks](#submitting-and-monitoring-tasks)
- [Handling Task Failures](#handling-task-failures)
- [Monitoring with Flower](#monitoring-with-flower)
- [Deployment in Poridhi Virtual Lab](#deployment-in-poridhi-virtual-lab)
- [Conclusion](#conclusion)

## Introduction

Task management is a crucial aspect of modern web applications that need to handle resource-intensive operations without blocking the main application flow. Asynchronous task queues solve this problem by offloading time-consuming tasks to worker processes that run in the background.

![celery workflow](https://github.com/Jilan5/poridhi---task-management-queue-using-celery-redis-and-flask/blob/main/images/celery%20workflow.drawio%20(4).svg)
### What is a Task Queue?

A task queue is a mechanism that distributes work across threads or machines. Task queues receive tasks with their associated data, execute them, and deliver the results. They can handle thousands of tasks per second and scale easily by adding more worker processes.

### Benefits of Task Queues

- **Improved User Experience**: Users don't have to wait for long-running operations to complete
- **Better Resource Management**: Distribute workload across multiple workers or machines
- **Scheduling Capabilities**: Execute tasks at specific times or intervals
- **Resilience**: Retry failed tasks automatically
- **Scalability**: Scale horizontally by adding more worker instances

### Real-world Examples

1. **Email Sending**: Instead of making users wait while your application sends emails, you can queue the emails to be sent in the background.
   
   ```python
   # Without task queue
   def register_user(user_data):
       create_user(user_data)
       send_welcome_email(user_data['email'])  # Could take seconds!
       return "Registration successful"
   
   # With task queue
   def register_user(user_data):
       create_user(user_data)
       send_welcome_email.delay(user_data['email'])  # Returns immediately
       return "Registration successful"
   ```

2. **Report Generation**: Generate complex reports in the background.

   ```python
   # Queue a report generation task
   @app.route('/generate-report')
   def generate_report():
       report_task = create_report.delay(user_id=current_user.id)
       return f"Report generation started. Task ID: {report_task.id}"
   ```

3. **Image Processing**: Handle resource-intensive operations like image resizing.

   ```python
   @celery.task
   def process_image(image_path):
       # Load image
       # Apply filters
       # Resize
       # Save processed image
       return processed_image_path
   ```
## Task Description of this Tutorial
In this tutorial, we'll set up a Flask application with Celery using Redis as the message broker and backend. This powerful combination enables you to build scalable web applications that can handle complex background tasks efficiently.

## Prerequisites

- Python 3.8 or later
- Docker (for running Redis)
- Basic understanding of Flask
- Linux/Unix environment or WSL on Windows

##Step 1: Setting Up the Environment

First, let's update our system and install the required packages:

```bash
sudo apt update
apt install python3.8-venv
```

Now, let's set up Redis using Docker:

```bash
docker run -p 6379:6379 --name some-redis -d redis
```

Verify that Redis is running:

```bash
docker exec -it some-redis redis-cli ping
```

You should see `PONG` as a response, indicating that Redis is working correctly.

Next, let's create a project directory and set up a virtual environment:

```bash
mkdir flask-celery-project && cd flask-celery-project
python3 -m venv env
source env/bin/activate
```

Create a requirements file with the necessary dependencies:

```bash
cat > requirements.txt << EOF
Flask==3.0.3
celery==5.4.0
redis==5.0.8
flower==2.0.1
EOF
```

Install the dependencies:

```bash
pip install -r requirements.txt
```

##Step 2: Creating a Simple Flask-Celery Application

Let's create a basic Flask application with Celery integration. Create a new file called `app.py`:

```python
from celery import Celery
from flask import Flask

app = Flask(__name__)

celery = Celery(
    __name__,
    broker="redis://127.0.0.1:6379/0",
    backend="redis://127.0.0.1:6379/0"
)


@app.route("/")
def hello():
    return "Hello, World!"


@celery.task
def divide(x, y):
    import time
    time.sleep(5)  # Simulating a time-consuming task
    return x / y
```

Here's what's happening in this file:

1. We create a Flask application instance.
2. We create a Celery instance, configuring:
   - Redis as the message broker (where tasks are queued)
   - Redis as the result backend (where task results are stored)
3. We define a simple route for our Flask application.
4. We create a Celery task called `divide` that simulates a long-running task with a 5-second sleep.

##Step 3: Running Celery Worker

Now, let's start our Celery worker to process tasks. Open a new terminal and run:

```bash
cd flask-celery-project
source env/bin/activate
celery -A app.celery worker --loglevel=info
```

You should see output indicating that the Celery worker has started successfully and is ready to process tasks.

##Step 4: Submitting and Monitoring Tasks

Let's interact with our application using the Flask shell. Open another terminal and run:

```bash
cd flask-celery-project
source env/bin/activate
FLASK_APP=app.py flask shell
```

In the Flask shell, let's import and execute our task:

```python
>>> from app import divide
>>> task = divide.delay(1, 2)
```

What's happening behind the scenes:

1. `divide.delay(1, 2)` sends a message to the Redis queue.
2. The Celery worker picks up this task from the queue.
3. The worker executes the task (waiting 5 seconds, then dividing 1 by 2).
4. The result (0.5) is stored in Redis.

Let's examine the task object:

```python
>>> type(task)
<class 'celery.result.AsyncResult'>
```

The `task` variable is an `AsyncResult` instance, which allows us to check the task's state and retrieve its result. Let's try to get the state and result:

```python
>>> print(task.state, task.result)
PENDING None
```

Initially, the task is in the `PENDING` state and has no result. If you keep checking, after about 5 seconds, you should see:

```python
>>> print(task.state, task.result)
SUCCESS 0.5
```

Now the task has completed successfully, and the result (0.5) is available.

##Step 5: Handling Task Failures

Let's see what happens when a task fails, for example, when trying to divide by zero:

```python
>>> task = divide.delay(1, 0)
>>> # Wait a few seconds
>>> task.state
'FAILURE'
>>> task.result
ZeroDivisionError('division by zero')
```

When a task fails, Celery captures the exception and stores it in the result backend. You can retrieve the exception from the `result` attribute of the `AsyncResult` object.

##Step 6: Monitoring with Flower

Flower is a web-based tool for monitoring Celery tasks and workers. Let's start Flower in a new terminal:

```bash
cd flask-celery-project
source env/bin/activate
celery -A app.celery flower --port=5555
```

###Step 7: Accessing Flower in Poridhi Virtual Lab

First, find your network interface's IP address:

```bash
ifconfig
# or
ip addr show
```

Look for the `eth0` interface and note the IP address.

In the Poridhi Virtual Lab, create a load balancer with your IP address and port 5555.

You can now access the Flower dashboard through the load balancer URL. If you're running this locally, you can access it at http://localhost:5555.

###Step 8: Using Flower to Monitor Tasks

The Flower dashboard shows:

1. Active workers
2. Queued tasks
3. Task history
4. Task details including:
   - UUID (task ID)
   - State
   - Arguments
   - Runtime
   - Result
![flower](https://github.com/Jilan5/poridhi---task-management-queue-using-celery-redis-and-flask/blob/main/images/flower%20dashboard.png?raw=true)
To examine a specific task using its UUID from the Flower interface:

```python
>>> from celery.result import AsyncResult
>>> task = AsyncResult('b09f126b-5c68-4695-a0f8-da77c88e9c0d')  # replace with your UUID
>>> task.state
>>> task.result
```


## Conclusion

In this tutorial, we've set up a Flask application with Celery and Redis for asynchronous task processing. We've learned:

- How to configure Celery with Redis as both the broker and backend
- How to create and execute tasks asynchronously
- How to monitor task execution and retrieve results
- How to handle task failures
- How to monitor Celery using Flower

This setup provides a solid foundation for building applications that need to handle time-consuming operations without blocking the main application flow. You can extend this by adding more complex tasks, scheduling periodic tasks, and implementing error handling and retry mechanisms.

For more information, refer to the official documentation:
- [Flask](https://flask.palletsprojects.com/)
- [Celery](https://docs.celeryq.dev/en/stable/)
- [Redis](https://redis.io/documentation)
- [Flower](https://flower.readthedocs.io/en/latest/)
