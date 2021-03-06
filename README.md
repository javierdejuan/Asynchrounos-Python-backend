# Asynchrounos-Python-backend
Building an Asynchrounos Python backend to process speech transcriptions requests from http.

## Requirements
A random user asks through an html front end to transcribe video speech.The user will send a Youtube URL as well as the language used in the video. 

## Architecture
As a transcription task may take a considerable amount of time, it is suitable to build an asynchrounos process.
For that purpose, we need a system to process asynchronous requests from html and a system to process transcription jobs.
For the first requirement a websocket service can be a good solution. Then, this websocket server will push into a queue the transcription requests.
Then, we need to use a queue manager to send jobs to the workers.
Finaly we need to deploy the system in production, under an Ubuntu 18.04 operating system.

## The Websocketservice
I choose websocket library from Python, which need asyncio modules to run. Below you can find a pattern script.

```Python
# WS server example
from rq import Queue
from redis import Redis
import asyncio
import websockets
import time
import json
from <myJobFunctions> import *

def processJob(inputfile,language,EsIndex,smartvideo_id):

    # we are going to use RQ Redis to push jobs in a queue
    # Tell RQ what Redis connection to use
    
    redis_conn = Redis()

    # get an instance of the queue
    q = Queue(connection=redis_conn)
 
    # enqueue transcription job
    
    job = q.enqueue(transcribe,inputfile,language)
    
    # enqueue ElasticSearch Injection job
    
    job = q.enqueue(ElasticInjection,EsIndex)
    
      return

async def hello(websocket, path):
    try:

        # waitingfor an http request
        
        message = await websocket.recv()
        
        # process the request, parse arguement and send it to process
        
        data =json.loads(message)
        inputfile=data['file']
        language=data['language']
        EsIndex=data['ESIndex']
        smartvideo_id=data['smartvideo_id']
        
        # proceessJob
        processJob(inputfile,language,EsIndex,smartvideo_id)
        
        # notify webclientsocket that the jobs are enqueued
        
        await websocket.send(f"[{time.ctime()}] webserverfunction:queued recognition process for file:{inputfile}, language:{lang$


    except Exception as ex:
        print(ex)
    return
    
# configure your websocket port and IP, and tell websocket what is your function handler, in my case is "hello"

start_server = websockets.serve(hello, <your server IP>,<your server socket port>,ping_timeout=None)

# loop

asyncio.get_event_loop().run_until_complete(start_server)
asyncio.get_event_loop().run_forever()



    
```

So far, we have a nice Python script which enables an asynchronous websocketserver. Unfortunaly, my jobs cannot run in asynchronous mode, since I am calling third party modules and they do not support this functionality. This implies that every websocket.recv() -> sync job -> websocket.send() block behaves in a synchronous way. 
To avoid this problem, instead of processing each job in the handler function, I delegate them pushing the job in a queue. Since, queueing process is very fast, we can then have a pseudo-asynchronous behaviour.

The enqueue process is done via [Redis](https://redis.io/). I decided to choose Redis because it is writting in C, it is light and fast, and it is optimized for jobbing Python functions. 

To deploy this websocket service in production and avoiding to manually execute the script, we are going to build a systemd service wrapper:
Go to  your ```var/systemd/system``` directory and create the service file, like this one:

```

Description="websocket service queue for mySmartlab"

[Service]
WorkingDirectory=/var/speechrecognition/

ExecStart=/var/speechrecognition/googlespeech/bin/python3 -u /var/speechrecognition/websocketserverqueue.py
Restart=on-failure


[Install]
WantedBy=multi-user.target

```
name it ```webServiceQueue.service```.

As I am using a Python Environments, we need to tell the service which is the appropiate Python version. The easiest way is to copy the directory from your environment into the ExecStart section of the service (In my case is ```/var/speechrecognition/googlespeech/bin/python3 ```)

The service should start under the following command: ``` service webServiceQueue start```, you can stop it with ``` service webServiceQueue stop``` and to follow the entry into the journal log, type ```journalctl -u webServiceQueue -f```.
In a developmente environnement, remember to execute ````systemctl daemon-reload```` every time you change the webServiceQueue.service file.

(be sure to add the ```Python -u``` option to have real time logging in ```journalctl``` entry)

So far we have a systemd service which handles Http requests and delegate the jobs via a Redis queue. For testing purposes, just pick up 
any WebSocketClientTester from the internet, enter your websocket service credentials and send a message. In my case, I have used a chrome extension in my browser like [this](https://chrome.google.com/webstore/detail/simple-websocket-client/pfdhoblngboilpfeibdedpjgfnlcodoo?hl=en) one.

Now, let's head the worker side of the process.

### The Worker Service

To execute jobs by a worker in Redis, just type ```rq worker``` from your directory. This will execute jobs in a worker thread.
The magic of Redis helps us to detach queueing process from job execution process. You will have an ouput like this:
<<< rq worker >>>

The last step is quite straight forward. We need to create another systemd service to host the ```rq worker``` command:

```

[Unit]
Description="rq worker for GitHub blog"

[Service]
WorkingDirectory=/var/www/html/mysmartlab/

ExecStart=/var/speechrecognition/googlespeech/bin/rq worker --path /var/speechrecognition/
Restart=on-failure

[Install]
WantedBy=multi-user.target


```
Type a rqworker.service file and fill in it. The ```--path``` option is used to "include" the directory of your Python modules to solve dependencies. In the other hand, the WorkingDirectory statement is the directory where the jobs are executed.  

### Conclusions

This tutorial shows an elegant and light Python backend which enables asynchronous behavour in you web application. Websocketservices also can be replaced with API REST. It has been done keeping things simple and clear. Of course, there are many other solutions to this problem, but here I have solved it using only the necessary elements avoiding a huge installation of external modules.  
