\# Project 2 API: Surface Defect Classifier + Inspection Note Analyzer



A FastAPI service that serves my Week 6 image model (ResNet18 on the NEU surface defect dataset, 6 classes) together with a text-evidence step that masks personal data in an inspector's note and decides REFER or ROUTINE.



\## How to run



```

python -m venv C:\\p2env

C:\\p2env\\Scripts\\activate

pip install torch torchvision --index-url https://download.pytorch.org/whl/cpu

pip install fastapi uvicorn python-multipart pillow pytest httpx numpy

uvicorn app:app

```



Then open http://127.0.0.1:8000/docs and use "Try it out".



\## Settings (in app.py)



\- MODEL\_PATH: week6\_model.pt

\- POSITIVE (class I must not miss): inclusion (the weakest class in Week 6, recall 0.983)

\- THRESHOLD: 0.5

\- URGENT words: crack, fracture, corrosion, spreading, severe, deep

\- Upload limit: 5 MB; note limit: 2,000 characters



\## API usage



| Endpoint | What it does |

|----------|--------------|

| GET /health | Shows the service is up, model loaded, the 6 classes and the POSITIVE class |

| POST /predict | Upload a JPEG or PNG; returns label, probabilities, latency\_ms |

| POST /analyze | Upload an image plus a note; returns prediction, masked note, urgent terms, decision, reasons, disclaimer |



Example calls:



```

curl -X POST http://127.0.0.1:8000/predict -F "file=@sample.jpeg;type=image/jpeg"

curl -X POST http://127.0.0.1:8000/analyze -F "file=@sample.jpeg;type=image/jpeg" -F "note=Visible crack spreading near the edge. Guardian 0300-1234567"

```



Example result for /predict on sample.jpeg (scratches\_257.jpg from the validation set): label = scratches, P(scratches) = 0.9794.



Example result for /analyze: the phone number and CNIC in the note came back as \[PHONE] and \[CNIC], urgent\_terms = crack, spreading, decision = REFER.



\## Error handling



| Bad input | Status | Response |

|-----------|--------|----------|

| Text file instead of image | 415 | Upload a JPEG or PNG image |

| Image over 5 MB | 413 | Image larger than 5 MB |

| File that is not a readable image | 400 | File is not a readable image |

| Missing file field | 422 | validation error |

| Note over 2,000 characters | 422 | Note longer than 2,000 characters |



None of these returns a stack trace.



\## Tests



`pytest -q test\_api.py` result:



```

8 passed, 1 warning in 21.52s

```



The tests cover health, schema and probabilities, parity with the direct model, wrong file type (415), too-large file (413), corrupt image (400), missing file (422) and no personal data leaking from /analyze. The one warning is a harmless httpx deprecation notice from the test client.



\## Latency benchmark



Hardware: Intel Core i5-8365U @ 1.60 GHz, 15.8 GB RAM, CPU only, 1 worker (Python 3.15.0, torch 2.15.0+cpu).



`python bench.py` (50 requests to /predict): p50 = 94 ms, p95 = 100 ms, max = 323 ms, throughput about 10.2 requests/s.



Little's law: with one worker at p50 = 94 ms, one worker handles about 1 / 0.094 = 10.6 requests/s, which is about 86,400 / 0.094 = 919,000 requests per day.



\## Bad input beyond the tests



I sent /analyze a note of 2,001 characters. The API returned HTTP 422 with the message "Note longer than 2,000 characters", with no crash. I also sent a 6 MB file (got 413) and a Python file labelled text/plain (got 415).



\## Check your understanding



1\. A PDF sent to /predict should get 415 (Unsupported Media Type). It is the client's mistake, so it must be a 4xx code. A 500 means a bug in the server, and here the server handles it on purpose without a stack trace.

2\. Loading the model takes seconds and uses memory. If it were loaded inside the endpoint, every request would pay that cost. In lifespan it loads once at startup and all requests reuse it.

3\. A median of 40 ms with a p95 of 400 ms means most requests are fast but a few are about 10 times slower. I would check the first request after startup, garbage collection, other programs using the CPU, queueing, and unusually large inputs.

4\. 500,000 requests per day is about 5.8 per second on average. The peak is 5 times that, about 29 per second. At 25 per second per worker, 29 / 25 = 1.16, so I need 2 workers.

5\. An async endpoint runs on a single event loop. Heavy model inference inside it blocks the loop, so other requests (even /health) have to wait. run\_in\_threadpool moves the heavy work to another thread.

6\. A 4-person team has 4 x 3 / 2 = 6 communication channels.

