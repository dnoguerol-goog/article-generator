# Article Generator Demo

## Overview
This is a simple demo that takes a snippet of article text, generates some metadata via Google's [Natural Language API](https://cloud.google.com/natural-language) and uploads an HTML file to a [Google Cloud Storage](https://cloud.google.com/storage) (GCS) bucket.

It is comprised of two microservices: `analyze-article-text` and `create-article-object`.

The goal of the demo is to facilitate demonstration a number of capabilities:

1. Two containerized microservices written in different languages that can be deployed to [Anthos Service Mesh](https://cloud.google.com/anthos/service-mesh) (ASM) or [Cloud Run](https://cloud.google.com/run).
1. Demonstrate interaction between microservices as either synchronous HTTP or asynchronous Pub/Sub.
1. Demonstrate traffic shaping, blue-green deployments, etc.
1. For ASM deployment, demonstrate fault injection.
1. For Cloud Run deployment, demonstrate [Events for Cloud Run](https://cloud.google.com/run/docs/quickstarts/events).
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

## Demo Topics

### TOPIC: Building the container images with Cloud Build

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
- A GCP IAM service account that has the TODO appropriate roles.

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
- A GCS bucket

#### Steps
1. In the `create-article-object` directory, update the `k8s/istio/create-article-object.yaml` file with the following: 

   a. The `Deployment` object's container image (around line 36) is set to the appropriate container registry URL.
   
   b. The `Deployment` object's PROJECT environment variable to your GCP project.
   
   c. The `Deployment` object's BUCKET environment variable to your GCS bucket (see requirements).

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

### TOPIC: Show ASM Topology, Cloud Logging and Cloud Trace

#### Requirements
- Successful generation of an HTML article from the previous step

#### Steps
1. Go into ASM and show details for each service.

2. Note the ability to create SLOs and alerts for services.

3. Show the ASM topology tab.

4. Go into Cloud Logging and show the log messages for the services. An example filter:

        resource.labels.project_id="$PROJECT_NAME"
        resource.labels.cluster_name="cluster-1"
        resource.type="k8s_container"
        resource.labels.namespace_name="demo"

5. Go into Cloud Trace and note the latest invocation and the correlation of the spans across services.

### TOPIC: Show fault injection in analyze-article-text service

#### Requirements
- Successful deployment of `analyze-article-text` service

#### Steps
1. Edit the `k8s/istio/analyze-article-text.yaml` file to change the `fault.abort.percentage.value` to `50` and the `fault.delay.percentage.value` to `50`. Save the file.

2. Update the deployment:

        kubectl -n demo apply -f k8s/istio/analyze-article-text.yaml

3. Call the service's `/version` endpoint to show the injected fault behavior:

        curl -v http://EXTERNAL-IP/version

### TOPIC: Deploy a second version of analyze-article-text service to ASM

#### Requirements
- Successful deployment of `analyze-article-text` service

#### Steps
1. If fault injection was demonstrated, edit the `k8s/istio/analyze-article-text.yaml` file to change the `fault.abort.percentage.value` to `0` and the `fault.delay.percentage.value` to `0`. Save the file and update the deployment:

        kubectl -n demo apply -f k8s/istio/analyze-article-text.yaml

2. Deploy the new version of the service:

        kubectl -n demo apply -f k8s/istio/analyze-article-text-v2.yaml

3. Show that the curl command still hits the 0.0.1 version:

        curl -v http://EXTERNAL-IP/version

4. Edit the `k8s/istio/analyze-article-text.yaml` file and change the weighting of v1 to 50 and the weighting of v2 to 50.

5. Apply the changes:

        kubectl -n demo apply -f k8s/istio/analyze-article-text.yaml

6. Show that the two versions are being load balanced:

        curl -v http://EXTERNAL-IP/version

### TOPIC: Deploy service to Cloud Run

#### Steps
1. Create the `create-article-object` service to Cloud Run:

    **Deployment platform:** Cloud Run (fully managed), region _europe-west4_  
    **Service name:** create-article-object  
    **Container image URL:** Your container image (0.0.1 version)  
    **Advanced Settings/General/Service account:** Your GCS service account  
    **Advanced Settings/Variables:** Create PROJECT variable with a value of your GCP project  
    **Advanced Settings/Variables:** Create BUCKET variable with a value of your GCS bucket (e.g. dn-text-entities)  
    **Ingress:** Allow all traffic  
    **Authentication:** Require authentication  
    **Additional trigger:**  
        Cloud Pub/Sub topic  
        Your GCS service account  
        `/events` URL path  

2. After service is created, click topic name on the triggers tab and then copy the topic name in the “Topic Details” screen. Be sure to remove the ``/projects/$PROJECT/topics`` prefix.

3. Create the `analyze-article-text` service to Cloud Run:

    **Deployment platform:** Cloud Run (fully managed), region _europe-west4_  
    **Service name:** analyze-article-text  
    **Container image URL:** Your container image (0.0.1 version)  
    **Advanced Settings/General/Service account:** Your GCS service account  
    **Advanced Settings/Variables:** Create PROJECT variable with a value of your GCP project  
    **Advanced Settings/Variables:** Create TOPIC variable with a value that was copied from step 2.  
    **Ingress:** Allow all traffic  
    **Authentication:** Require authentication

4. When the service is finished deploying, copy the URL that is generated for the service.

5. Post an article:

        curl -v -H "Authorization: Bearer $(gcloud auth print-identity-token)" -d 'This is a test article.' $SERVICE_URL/publish

6. Verify that the HTML file appears in the GCS bucket.

### TOPIC: Deploy service v2 to Cloud Run

TODO
