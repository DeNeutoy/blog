

## TIL: Google Cloud Run with Eventarc Triggers

Today, I wanted to set up a Google Cloud Run API endpoint which was called everytime another service uploaded something into a Google Cloud Storage bucket.

### Attempt 1

It is possible to set up a Cloud Run deployment which is deployed from a trigger rather than serving a HTTP endpoint. I did this in the UI, selecting the type of event (`google.cloud.storage.object.v1.finalized`). However, the UI for this seems to be in Beta, which meant the filtering is restricted to a single cloud storage object, which for my use was completely pointless. I needed to be able to specify a bucket, so that the API wasn't called for every single object creation call in GCS. I couldn't do this, so just to see if it worked, I tried it without any limit.

This worked, but caused Something Very Bad - the Cloud Run API wrote logs to GCS every time it was invoked, but every time it wrote one of the log files, it triggered itself again, because an object was being written to GCS. Bad Bad.

### Attempt 2

After some digging I figured out that creating triggers from the commandline is more flexible. So I deleted the trigger I attached to the Cloud Run deployment and created a new one.

Created trigger using:

```
gcloud eventarc triggers create $TRIGGER_NAME\
  --destination-run-service=eventarc-test \
  --destination-run-region=us-central1 \
  --event-filters="type=google.cloud.storage.object.v1.finalized" \
  --event-filters="bucket=$BUCKET_NAME" \
  --service-account=<my service account>@developer.gserviceaccount.com \
  --location=us
```

Before this would work, I had to add Pub/Sub publisher permissions to the IAM role for GCS:

```
gcloud projects add-iam-policy-binding <my gcp project name> --member="serviceAccount:${SERVICE_ACCOUNT}" \
    --role='roles/pubsub.publisher'
```

Additionally, there were a couple of gotchas:

- The bucket that is creating the trigger has to be in the same location as the `--location` argument of the trigger.
- Cloud Run invocations triggered by a trigger rather than via HTTP have a max running time of 15 min vs 1hr normally (this is not doccumented anywhere and is super annoying).
- Certain event filters seem to be invalid in certain regions. Initially I tried to create the trigger with `--location=global` but got the following error:

```
ERROR: (gcloud.eventarc.triggers.create) INVALID_ARGUMENT: The request was invalid: [INVALID_ARGUMENT] The request was invalid: invalid value for attribute 'type' in trigger.event_filters: type "google.cloud.storage.object.v1.finalized" is not allowed in global location
```

### Once it is working

Once I actually got it working, you have to look at the request headers to find the info about what is being sent via the trigger. Basically all of the content is contained within the `ce-*` headers, which stand for [CloudEvents](https://cloud.google.com/eventarc/docs/cloudevents).


```python

from fastapi import FastAPI, Request

app = FastAPI()

@app.post("/")
def root(request: Request):

    print(request.headers)
    # Example response
    {
        'host': 'eventarc-test-s6gfdsap7fda-uc.a.run.app',
        'content-type': 'application/json',
        'content-length': '838',
        'accept': 'application/json',
        'from': 'noreply@google.com',
        'user-agent': 'APIs-Google; (+https://developers.google.com/webmasters/APIs-Google.html)',
        'x-cloud-trace-context': '483010b253289e906d9324f80bad0cd3/9989356814841849077;o=1',
        'traceparent': '00-483010b253389e906d9324f80bad0cd3-8aa15318e346a4f5-01',
        'x-forwarded-for': '66.112.6.102',
        'x-forwarded-proto': 'https',
        'forwarded': 'for="66.112.6.102";proto=https',
         'accept-encoding': 'gzip, deflate, br',
         'ce-id': '33212954fdsa24407',
         'ce-source': '//storage.googleapis.com/projects/_/buckets/mappable-upload-data-dev',
         'ce-specversion': '1.0',
         'ce-type': 'google.cloud.storage.object.v1.finalized',
         'ce-subject': 'objects/<object name in storage>',
         'ce-time': '2021-11-02T11:10:56.505568Z',
         'ce-bucket': '<bucket name>'
         }
```

Having not done this type of architecture design before I'm finding it a little bizzare how difficult it is to set up an app which can trigger events and run tasks asyncronously. There are plenty of tools for this (Celery etc), but deploying them seems tricky.





