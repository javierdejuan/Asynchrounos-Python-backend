# Asynchrounos-Python-backend
building an Asynchrounos Python backend to process speech transcriptions requests from http

## Requirements
A random user asks through an html front end to transcribe video speech.

## Architecture
As the transcription task may take a considerable amount of time, it is suitable to build an asynchrounos process.
For that purpose, a websocket service should be available. This websocket server will push into a queue the transcription requests.
Then, we need to use a queue manager to send jobs to the workers.
Finaly we need to deploy two services under systemd management:
* socketqueue.service
* rqworker.service

