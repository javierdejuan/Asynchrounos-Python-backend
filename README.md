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
I choose websocket library from Python, which need asyncio modules to run.
```
# WS server example
from rq import Queue
from redis import Redis
import asyncio
import websockets
import time
import json
from transcribe import *
from transcribeFLAC import *
from bulkInjectionES import *

def processJob(inputfile,language,EsIndex,smartvideo_id):

    # Tell RQ what Redis connection to use
    
    redis_conn = Redis()

    q = Queue(connection=redis_conn)
 
    # enqueue transcription job
    
    job = q.enqueue(transcribefromURL,inputfile,language)
    
    # enqueue ElasticSearch Injection job
    
    job = q.enqueue(ElasticInjection)
    
    
```
