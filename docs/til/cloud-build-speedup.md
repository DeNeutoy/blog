
## TIL: Speeding up Google Cloud Build by an order of magnitude

[Cloud Build](https://cloud.google.com/build) is a cool tool for building docker images, which integrates very nicely with other Google services, allowing you to deploy code automatically. I am using it extensively to build the frontend and backend components of a project I am working on, which is super helpful. Every time I push a commit to github, Cloud Build is triggered, builds all my Docker images and deploys them automatically. It's an extremely smooth workflow.

One issue with Cloud Build is that the product managers for the service at Google are in a bit of a bind. They have no incentive to provide methods to users to make their builds efficient, because all of their billing is based on build time. This is a bit of a difficult situation, as it means that by default, some Google services with "autodeploy" features ([Cloud Run](https://cloud.google.com/run) in particular) do not use any of the caching features of Docker. This is one of the main benefits of Docker! It's a little frustrating that caching in builds is not enabled by default for other Google services which use Cloud Build in the background to deploy your code. Like, why not?

### The initial CloudBuild file:

```yaml
steps:
  - name: gcr.io/cloud-builders/docker
    args:
      - build
      - '--no-cache'
      - '-t'
      - '$_GCR_HOSTNAME/$PROJECT_ID/$REPO_NAME/$_SERVICE_NAME:$COMMIT_SHA'
      - api
      - '-f'
      - api/Dockerfile
    id: Build
  - name: gcr.io/cloud-builders/docker
    args:
      - push
      - '$_GCR_HOSTNAME/$PROJECT_ID/$REPO_NAME/$_SERVICE_NAME:$COMMIT_SHA'
    id: Push
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk:slim'
    args:
      - run
      - services
      - update
      - $_SERVICE_NAME
      - '--platform=managed'
      - '--image=$_GCR_HOSTNAME/$PROJECT_ID/$REPO_NAME/$_SERVICE_NAME:$COMMIT_SHA'
      - >-
        --labels=managed-by=gcp-cloud-build-deploy-cloud-run,commit-sha=$COMMIT_SHA,gcb-build-id=$BUILD_ID,gcb-trigger-id=$_TRIGGER_ID,$_LABELS
      - '--region=$_DEPLOY_REGION'
      - '--quiet'
    id: Deploy
    entrypoint: gcloud
timeout: 1200s
images:
  - '$_GCR_HOSTNAME/$PROJECT_ID/$REPO_NAME/$_SERVICE_NAME:$COMMIT_SHA'
options:
  substitutionOption: ALLOW_LOOSE
substitutions:
  _LABELS: gcb-trigger-id=c21fdsf16c-373a-41b9-9d80-7fdsffs42d219bc8
  _TRIGGER_ID: c21fdsfs16c-373a-4fdsfdsb9-9d80-7d342d2fdsfsdc8
  _DEPLOY_REGION: asia23
  _GCR_HOSTNAME: us.gcr.io
  _PLATFORM: managed
  _SERVICE_NAME: redacted
tags:
  - gcp-cloud-build-deploy-cloud-run
  - gcp-cloud-build-deploy-cloud-run-managed
```


### Speeding it up

Currently, the docker build step in the above cloudbuild file uses no cache. To speed this up, we are going to do 3 steps:

1. Pull the built image with a `latest` from Google Container Registry
2. Push the image we build with an additional tag, `latest`, so we can fetch it in the next build.
3. Pass the image we pulled as a cache to the build step.


### Step 1: Pulling an image

This step is easy - we just add a new step to our `cloudbuild.yaml` file which pulls an image from the Google Container Registry (or some other registry, like DockerHub).
```yaml
- name: gcr.io/cloud-builders/docker
  entrypoint: bash
  args:
    - '-c'
    - >-
      docker pull $_GCR_HOSTNAME/$PROJECT_ID/$REPO_NAME/$_SERVICE_NAME:latest
      || exit 0

```

### Step 2: Tagging it Twice

Next up - we need to modify the build step in our `cloudbuild.yaml` to do 2 things:

1. Tag the image we build with `:latest`  *as well as* the commit sha that we will use in production deployments.
2. Use the image we pulled in the previous step as a cache.

```yaml
- name: gcr.io/cloud-builders/docker
  args:
    - build
    - '-t'
    - '$_GCR_HOSTNAME/$PROJECT_ID/$REPO_NAME/$_SERVICE_NAME:$COMMIT_SHA'
    - '-t'
    - '$_GCR_HOSTNAME/$PROJECT_ID/$REPO_NAME/$_SERVICE_NAME:latest'
    - '--cache-from'
    - '$_GCR_HOSTNAME/$PROJECT_ID/$REPO_NAME/$_SERVICE_NAME:latest'
    - api
    - '.'
  id: Build
```

### Step 3: Pushing the `latest` tag

Finally, we need to push the image to GCR with the latest tag, as well as the image tagged with the commit sha, so the next build can just fetch the image taged with "latest".

```yaml
- name: gcr.io/cloud-builders/docker
  args:
    - push
    - '$_GCR_HOSTNAME/$PROJECT_ID/$REPO_NAME/$_SERVICE_NAME:latest'
  id: PushLatest
```

The result of adding these caching steps is a build time reduced from 19min (this is a particularly slow build, because it has to compile a Python library which uses a lot of C++ bindings) to 1min 40 sec. Not bad!


