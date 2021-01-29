# Article Generator Demo

## Overview
This is a simple demo that takes a snippet of article text, generates some metadata via Google's [Natural Language API](https://cloud.google.com/natural-language) and uploads an HTML file to a [Google Cloud Storage](https://cloud.google.com/storage) (GCS) bucket.

It is comprised of two microservices: `analyze-article-text` and `create-article-object`.

The goal of the demo is to facilitate demonstration of a number of capabilities:

1. Two containerized microservices written in different languages that can be deployed to [Anthos Service Mesh](https://cloud.google.com/anthos/service-mesh) (ASM) or [Cloud Run](https://cloud.google.com/run).
1. Demonstrate interaction between microservices as either synchronous HTTP or asynchronous Pub/Sub.
1. Demonstrate traffic shaping to support blue-green deployments, canaries, etc.
1. For ASM deployment, demonstrate fault injection and multiple service versions.
1. For Cloud Run deployment, demonstrate [Events for Cloud Run](https://cloud.google.com/run/docs/quickstarts/events) and multiple service revisions.
1. Demonstrate distributed tracing between the two microservices.

## Caveat

This demo is not intended to demonstrate best practices for implementing microservices or optimal usage of GCP services nor should it be treated as such. It has purposely been kept reasonably simple to facilitate quick demo deployments demonstrating various GCP capabilities.

## Services

### analyze-article-text

A Go service that calls the natural language API to perform analysis of the article text.

#### Building

A [Cloud Build](https://cloud.google.com/cloud-build) file is included in the service root directory for building the container image using [Cloud Native Buildpacks](https://buildpacks.io/) because, you know, why we would want to create Dockerfiles? ;-)

#### Environment variables

| Name      | Description                              |
|-----------|----------------------------------------- |
| `PROJECT` | The GCP project                          |
| `PORT`    | The port to listen on (defaults to 8080) |
| `SERVICE_HOST` | The hostname of the `create-article-object` to POST results to. This can optionally include the port number (e.g.  `localhost:8081`). Please note that either `SERVICE_HOST` or `TOPIC` should be used - not both.|
| `TOPIC` | A [Cloud PubSub](https://cloud.google.com/pubsub) topic to publish results to. If you are using a topic automatically generated by Events for CloudRun, be sure to exclude the `/projects/$PROJECT/topics` prefix. Please note that either `TOPIC` or `SERVICE_HOST` should be used - not both. |

#### Endpoints

##### `/publish` 

Accepts a POSTed text string. For example:

    curl -v -d 'This is some article text' http://localhost:8080/publish

The text string will be sent to the Natural Language API to extract text entities and perform sentiment analysis. 

A JSON document will then be created with the article text, entites and sentiment results.

This will then be sent to the `create-article-object` service via:

- An HTTP POST (if the `SERVICE_HOST` environment variable is defined). In this case, the service will automatically wrap the JSON document in CloudEvent JSON.

- A GCP PubSub topic (if the `TOPIC` environment variable is defined).

##### `/version`

Accepts a GET requests and returns the version number of the service as a text string.

This endpoint is helpful for testing, showing traffic shaping and deployment updates.

#### Distributed Tracing

The service creates custom spans using OpenTracing to demonstrate distributed tracing. These will automatically be sent to Cloud Trace.

### create-article-object

A Node.js service that receives a CloudEvent containing a JSON document with article text, article entities and a sentiment result. It will generate an HTML file from this data and upload it to a Google Cloud Storage bucket.

#### Building

A [Cloud Build](https://cloud.google.com/cloud-build) file is included in the service root directory for building the container image using [Cloud Native Buildpacks](https://buildpacks.io/) because, you know, why we would want to create Dockerfiles? ;-)

#### Environment variables

| Name      | Description                              |
|-----------|----------------------------------------- |
| `PROJECT` | The GCP project                          |
| `PORT`    | The port to listen on (defaults to 8080) |
| `BUCKET`  | The GCS bucket to upload to              |

#### Endpoints

##### `/events`

Accepts an HTTP POST containing a JSON document in CloudEvent format.

The service will then generate an HTML file (currently using an embedded template string) and create an HTML file in the configured GCS bucket. The filename will be in the form of

     RANDOM_UUID.html

##### `/version`

Accepts a GET requests and returns the version number of the service as a text string.

This endpoint is helpful for testing, showing traffic shaping and deployment updates.

#### Distributed Tracing

The service creates custom spans using OpenTracing to demonstrate distributed tracing. These will automatically be sent to Cloud Trace.

## Demonstration Topics

### TOPIC: Building the container images with Cloud Build

_NOTE: This is not necessary as the out-of-the-box K8s manifests will pull containers from Docker Hub._

#### Requirements
- Both the `analyze-article-text` and `create-article-object` directories have been added as separate GCP Cloud Source Repositories repos.

#### Steps
1. Create a new Cloud Build trigger for `analyze-article-text`:

    **Name:** analyze-article-text  
    **Event:** Select "Push new tag" radio button  
    **Source/Repository:** Your Cloud Source repo (see requirements)  
    **Source/Tag:** .*
   
   Then click "Create".

2. From the list of triggers, manually trigger the `analyze-article-text` trigger by clicking the **RUN** button next to its name, enter a tag of `0.0.1` and then click the **RUN TRIGGER** button.

3. Repeat the above two steps for the `create-article-object` service.

4. Go into Container Registry and confirm that `analyze-article-text` and `create-article-object` container images exist with `0.0.1` version tags.

5. Note that vulnerability scanning has started on the container images.

### TOPIC: Deploy analyze-article-text service to Anthos Service Mesh

#### Requirements
- A GKE cluster that has [Anthos Service Mesh](https://cloud.google.com/service-mesh/docs/install) installed and [Workload Identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity) enabled.
- A command-line with `kubectl` configured to talk to the cluster.
- A GCP IAM service account that has the appropriate roles (TODO: define).

#### Steps
1. Create a new namespace for the services:
     
        kubectl create ns demo 

2. Enable [ASM sidecar injection](https://cloud.google.com/service-mesh/docs/proxy-injection) for the namespace. Make sure you use the correct revision name (value of `rev=` in the below example) as mentioned in the previous link. For example:

        kubectl label namespace demo istio-injection- istio.io/rev=asm-181-5 --overwrite

3. In the `analyze-article-text` directory, update the `k8s/istio/analyze-article-text.yaml` file with the following: 

   a. The `ServiceAccount` object's `iam.gke.io/gcp-service-account` annotation (around line 7) is set the correct GCP service account name (as mentioned in the requirements section).

   b. The `Deployment` object's container image (around line 47) is set to the appropriate container registry URL.

   c. The `Deployment` object's PROJECT environment variable to your GCP project.

4. From the `analyze-article-text` directory, deploy the service:

        kubectl -n demo apply -f k8s/istio/analyze-article-text.yaml

   You should see confirmation that the objects were created:

        serviceaccount/demo created
        service/analyze-article-text-service created
        deployment.apps/analyze-article-text-v1-deployment created
        gateway.networking.istio.io/article-gateway created
        virtualservice.networking.istio.io/articles created
        destinationrule.networking.istio.io/articles created   

5. Bind the newly created K8s service account to the GCP service account using workload identity. Be sure to replace the GCP service account (last parameter) with yours.

        gcloud iam service-accounts add-iam-policy-binding --role roles/iam.workloadIdentityUser --member "serviceAccount:$PROJECT.svc.id.goog[demo/demo]" $SA_NAME@$PROJECT.iam.gserviceaccount.com

6. Verify the running pods and that there are two containers in it:

        kubectl -n demo get pods

7. Get the EXTERNAL-IP of the ingress gateway:

        kubectl -n istio-system get svc istio-ingressgateway

8. Run a curl command to check that the service is working:

        curl -v http://EXTERNAL-IP/version

### TOPIC: Deploy create-article-object service to ASM

#### Requirements
- A working `analyze-article-text` service from the previous section
- A GCS bucket that will store the generated HTML files

#### Steps
1. In the `create-article-object` directory, update the `k8s/istio/create-article-object.yaml` file with the following: 

   a. Set the value of the `Deployment` object's container image (around line 36) is set to the appropriate container registry URL.
   
   b. Set the value of the `Deployment` object's PROJECT environment variable to your GCP project.
   
   c. Set the value of the `Deployment` object's BUCKET environment variable to your GCS bucket (see requirements).

2. From the `create-article-object` directory, deploy the service:

        kubectl -n demo apply -f k8s/istio/create-article-object.yaml

   You should see confirmation that the objects were created:

        service/create-article-object-service created
        deployment.apps/create-article-object-deployment created

### TOPIC: Submit article text to ASM deployed services

#### Requirements
- A working `analyze-article-text` service from the previous section
- A working `create-article-object` service from the previous section

#### Steps
1. Get the EXTERNAL-IP of the ingress gateway:

        kubectl -n istio-system get svc istio-ingressgateway

2. Submit article text to the `analyze-article-text` service:

        curl -v -d 'This is a test article.' http://EXTERNAL-IP/publish

3. Check the GCS bucket for the new HTML file.

### TOPIC: Show additional ASM details

#### Requirements
- Successful generation of an HTML article from the previous step

#### Steps
1. Run some load on the `analyze-article-text` service:

        kubectl run httperf --rm -it --image=dos65/httperf -- httperf --server EXTERNAL_IP --uri /version --rate 50 --timeout 5 --num-conn=5000

2. Go into ASM and show details for each service.

3. Note the ability to create SLOs and alerts for services.

4. Show the ASM topology tab.

### TOPIC: Show Cloud Logging and Cloud Trace

1. Go into Cloud Logging and show the log messages for the services. An example filter (replacing $PROJECT_NAME and $CLUSTER_NAME appropriately):

        resource.labels.project_id="$PROJECT_NAME"
        resource.labels.cluster_name="$CLUSTER_NAME"
        resource.type="k8s_container"
        resource.labels.namespace_name="demo"

2. Go into Cloud Trace and note the latest invocation and the correlation of the spans across services.

### TOPIC: Show fault injection in analyze-article-text service

#### Requirements
- Successful deployment of `analyze-article-text` service

#### Steps
1. Edit the `k8s/istio/analyze-article-text.yaml` file to change the `fault.abort.percentage.value` to `50` and the `fault.delay.percentage.value` to `100`. Save the file.

2. Update the deployment:

        kubectl -n demo apply -f k8s/istio/analyze-article-text.yaml

3. Call the service's `/version` endpoint to observe the injected fault behavior:

        curl -v http://EXTERNAL-IP/version

Note that the response takes 2 seconds and returns an HTTP 503 error roughly half the time.

### TOPIC: Deploy a second version of analyze-article-text service to ASM

#### Requirements
- Successful deployment of `analyze-article-text` service

#### Steps

From the `analyze-article-text` directory:

1. If fault injection was demonstrated, edit the `k8s/istio/analyze-article-text.yaml` file to change the `fault.abort.percentage.value` to `0` and the `fault.delay.percentage.value` to `0`. Save the file and update the deployment:

        kubectl -n demo apply -f k8s/istio/analyze-article-text.yaml

2. Edit the `k8s/istio/analyze-article-text-v2.yaml` file with the following: 

   a. Set the value of the `Deployment` object's container image (around line 23) is set to the appropriate container registry URL.
   
   b. Set the value of the `Deployment` object's PROJECT environment variable to your GCP project.
   
   c. Set the value of the `Deployment` object's BUCKET environment variable to your GCS bucket (see requirements).

2. Deploy the new version of the service:

        kubectl -n demo apply -f k8s/istio/analyze-article-text-v2.yaml

3. Show that the curl command still hits the 0.0.1 version:

        curl -v http://EXTERNAL-IP/version

4. Edit the `k8s/istio/analyze-article-text.yaml` file and change the weight of v1 to `50` and the weighting of v2 to `50`.

5. Apply the changes:

        kubectl -n demo apply -f k8s/istio/analyze-article-text.yaml

6. Show that the two versions are being load balanced:

        curl -v http://EXTERNAL-IP/version

### TOPIC: Deploy services to Cloud Run

#### Steps
1. Download the container images locally and upload to Container Registry (replacing $PROJECT with your GCP project):

        docker pull googldan/create-article-object:0.0.1
        docker pull googldan/analyze-article-text:0.0.1
        docker pull googldan/analyze-article-text:0.0.2
        docker tag googldan/create-article-object:0.0.1 eu.gcr.io/$PROJECT/create-article-object:0.0.1
        docker tag googldan/analyze-article-text:0.0.1 eu.gcr.io/$PROJECT/analyze-article-text:0.0.1
        docker tag googldan/analyze-article-text:0.0.2 eu.gcr.io/$PROJECT/analyze-article-text:0.0.2

2. Create the `create-article-object` service to Cloud Run:

    **Deployment platform:** Cloud Run (fully managed), region _europe-west4_  
    **Service name:** create-article-object  
    **Container image URL:** `eu.gcr.io/$PROJECT/create-article-object:0.0.1`  
    **Advanced Settings/General/Service account:** Your GCS service account  
    **Advanced Settings/Variables:** Create PROJECT variable with a value of your GCP project  
    **Advanced Settings/Variables:** Create BUCKET variable with a value of your GCS bucket (e.g. dn-text-entities)  
    **Ingress:** Allow all traffic  
    **Authentication:** Require authentication  
    **Additional trigger:**  
        Cloud Pub/Sub topic  
        Your GCS service account  
        `/events` URL path  

3. After service is created, click topic name on the triggers tab and then copy the topic name in the “Topic Details” screen. Be sure to remove the ``/projects/$PROJECT/topics`` prefix.

4. Create the `analyze-article-text` service to Cloud Run:

    **Deployment platform:** Cloud Run (fully managed), region _europe-west4_  
    **Service name:** analyze-article-text  
    **Container image URL:** `eu.gcr.io/$PROJECT/analyze-article-text:0.0.1`  
    **Advanced Settings/General/Service account:** Your GCS service account  
    **Advanced Settings/Variables:** Create PROJECT variable with a value of your GCP project  
    **Advanced Settings/Variables:** Create TOPIC variable with a value that was copied from step 2.  
    **Ingress:** Allow all traffic  
    **Authentication:** Require authentication

5. When the service is finished deploying, copy the URL that is generated for the service.

6. Post an article:

        curl -v -H "Authorization: Bearer $(gcloud auth print-identity-token)" -d 'This is a test article.' $SERVICE_URL/publish

7. Verify that the HTML file appears in the GCS bucket.

### TOPIC: Deploy analyze-article-text v2 to Cloud Run

#### Requirements
- Successful deployment of `analyze-article-text` v1 service to Cloud Run

#### Steps
1. From Cloud Run, click into the existing `analyze-article-text` service.

2. Click the **EDIT & DEPLOY NEW REVISION** button.

3. For Container image URL, use `eu.gcr.io/$PROJECT/analyze-article-text:0.0.2`.

4. Scroll down and uncheck the checkbox labeled "Service this revision immediately".

5. Click the **DEPLOY** button.

6. Note the second revision is created with 0% traffic.

7. Show that traffic is still going only to 0.0.1:

        curl -v -H "Authorization: Bearer $(gcloud auth print-identity-token)" $SERVICE_URL/version

8. Click the overflow menu icon next to the new revision and click "Manage Traffic". Change the traffic percentage from 0 to 50 and click the **SAVE** button.

9. Show that traffic is split between 0.0.1 and 0.0.2 by executing curl a few times:

        curl -v -H "Authorization: Bearer $(gcloud auth print-identity-token)" $SERVICE_URL/version
