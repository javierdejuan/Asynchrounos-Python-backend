# Asynchrounos-Python-backend
building an Asynchrounos Python backend to process speech transcriptions requests from http

## Requirements
A random user asks through an html front end to transcribe video speech.

## Architecture
As a transcription task may take a considerable amount of time, it is suitable to build an asynchrounos process.
For that purpose, we need a system to process asynchronous requests from html and a system to process transcription jobs.
For the first requirement a websocket service can be a good solution. Then, this websocket server will push into a queue the transcription requests.
Then, we need to use a queue manager to send jobs to the workers.
Finaly we need to deploy two services under some kind of OS management, as supervisor, systemd or similars.

## The Wbsocketservice
I choose websocket library from Python, which need asyncio modules to run. Below you can find a pattern script.

```
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

start_server = websockets.serve(hello, <your server IP>,<your server socket port>,ping_timeout=None)
asyncio.get_event_loop().run_until_complete(start_server)

    
```


then in your ```var/systemd/system``` create the service file, like this one:

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

The service should start under the following command: ``` service webServiceQueue start```, you can stop it with ``` service webServiceQueue stop``` and to follow the entry into the journal log, type ```journalctl -u webServiceQueue -f```.
In a developmente environnement, remember to execute ````systemctl daemon-reload```` every time you change the webServiceQueue.service file.
