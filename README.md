# DeepSparse + Milvus: Text-Search with BERT

This example demonstrates how to create a text-search engine with a microservices architecture using FastAPI, DeepSparse, Milvus, and MySQL.

We will create 4 services:
- Milvus Server - vector database used to hold the embeddings of the dataset and perform the search queries
- MySQL Server - holds the mapping from Milvus Ids to original title,text pairs
- DeepSparse Server - inference runtime used to generate the embeddings for the queries
- Application Server - endpoint called by the client side with queries for searching

We will demonstrate running on a single local machine as well as in a VPC on AWS with independent-scaling of the App, Database, and Model Serving Components.

## Text-Search with BERT - How/Why it Works

Deep Learning models like BERT are useful for building search engines. BERT takes unstructured input like text and projects the text into a fixed-length vector (called an embedding). These embeddings are generally useful for encoding the semantic meaning of the original text, with interesting relationships like the embedding of `queen` being approximately equal to the embedding of `king - man + woman`.

Because these embeddings encode semantic meaning of text, they can be used to create a search engine over a document list. First, we project each document into the embedding space with BERT. Then, when a user provides a query, we project the query into the same embedding space with BERT. Once the query and document list are both in the embedding space, we can run a nearest neighbors algorithm to find the most semantically similiar examples!

#### Seems Simple Enough: Why Do We Need DeepSparse and Milvus?

While the ideas are quite simple, there is signficiant computation in two steps. DeepSparse and Milvus make it easy for users to handle the compute demands of these key functions.

**1. Generating the embeddings:** BERT is a large model + is slow to run. This has a big impact to the system design in two steps. First, generating the embeddings for the document databases (which can have millions or more documents) is an offline batch process which can take a long time and require significiant resources (especially if embeddings are updated consistently). Second, responding to queries in an online fashion is latency-senstive. Running a big model like BERT can dominate the query response time. Without DeepSparse, users often turn to specialized hardware like GPUs to accelerate the computation, complicating the deployment process.

- DeepSparse is CPU-only deep learning deployment platform. DeepSparse can reach GPU-class performance on inference-optimized sparse models. This means that users can achieve the level of performance needed for production without the burdens of running with specialized hardware.  Users can scale DeepSparse deployments vertically from 2 to 192 cores, tailoring the size of their footprint to their application’s specific needs. Users can scale DeepSparse horizontally with standard Kubernetes, including using managed Kubernetes services like GKE. Users can deploy the same models on any hardware from Intel to AMD to ARM and from cloud to data center to edge, including on pre-existing systems. With software-defined inference, companies simplify the model deployment process without compromising on performance, bringing more models to production and reducing overall TCO.

**2. Finding the closest vectors:** Running the naive algorithm to find the closest vectors in the embedding space is expensive. A user would have to compute the distance of the query to every element in the dataset and then run a sort algorithm `O(N*log(N))`. This is prohibative for a latency-senstive query. Without Milvus, users can only run at small scale or need to stand up complex state-of-the-art systems that can store and run the search faster. 

- Milvus with singular goal: store, index, and manage massive embedding vectors generated by deep neural networks and other machine learning (ML) models. As a database specifically designed to handle queries over input vectors, it is capable of indexing vectors on a trillion scale. Under the hood, Milvus creates [indexes](https://milvus.io/docs/v2.1.x/index.md) in a smart way and has efficient implmentations of Approximate nearest neighbor (ANN) search algorithms are used to accelerate the searching process. In such a way, Milvus makes it trivial to add vector search into an application.

## Application Architecture

We have provided a sample dataset in `server/data/example.csv`. These data are articles about various topics, in `(title,text)` pairs.

For each article, we project the `text` into the embedding space with BERT running on DeepSparse. We then store each vector embedding in Milvus with a unique primary key `id` and store the `(id, title,text)` tripes in MySQL. DeepSparse, Milvus, and MySQL are each independent servers with REST APIs exposed.

We have a app server built on FastAPI. The app server exposes a `/search` endpoint, which allows clients to send `text` to the server via `GET`. The app server sends the query `text` to DeepSparse Server, which returns the embedding of the query. The app server then sends the embedding to Milvus, which searches for the 10 most similiar vectors in the database and returns their `ids` to the app server. The app server then looks up the `title` and `text` of the IDs in MySQL and returns them back to the client.

As such, we have a microservices architecture and can scale the app server, databases, and model service independently.

## Running Locally

### Start the Server

#### Installation:
- Milvus and Postgres are installed using Docker containers. [Install Docker](https://docs.docker.com/engine/install/) and [Docker Compose](https://docs.docker.com/compose/install/linux/).
- DeepSparse is installed via PyPI. Create a virtual enviornment and run `pip install -r server/deepsparse-requirements.txt`.
- The App Server is based on FastAPI. Create a virtual enviornment and run `pip install -r server/app-requirements.txt`.

#### 1. Start Milvus

Milvus has a convient `docker-compose` file which can be downloaded with `wget` that launches the necessary services needed for Milvus. 

``` bash
cd server/database-server
wget https://raw.githubusercontent.com/milvus-io/milvus/master/deployments/docker/standalone/docker-compose.yml -O docker-compose.yml
sudo docker-compose up
cd ..

```
This command should create `milvus-etcd`, `milvus-minio`, and `milvus-standalone`.

#### 2. Start MySQL

MySQL can be started with the base MySQL image available on Docker Hub. Simply run the following command.

```bash
docker run -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 -d mysql:5.7
```

Our Databases have been started~

#### 3. Start DeepSparse Server

DeepSparse not only includes high performance runtime on CPUs, but also comes with tooling that simplify the process of adding inference to an application. Once example of this is the Server functionality, which makes it trivial to stand up a model service using DeepSparse.

We have provided a configuration file in `/server/deepsparse-server/server-config-deepsparse.yaml`, which sets up an embedding extraction endpoint running a sparse version of BERT from SparseZoo. You can edit this file to adjust the number of workers you want (this is the number of concurrent inferences that can occur). Generally, its a fine starting point to use `num_cores/2`.

Here's what the config file looks like.

```yaml
# num_cores: 8  # number of cores availble for use - defaults to all 
num_workers: 4  # number of streams - should be tuned, num_cores / 2 is good place to start

endpoints: 
  - task: embedding_extraction
    model: zoo:nlp/masked_language_modeling/bert-base/pytorch/huggingface/wikipedia_bookcorpus/pruned80_quant-none-vnni
    route: /predict
    name: embedding_extraction_pipeline
    kwargs:
      return_numpy: False
      extraction_strategy: reduce_mean
      sequence_length: 512
      engine_type: deepsparse

```

To start DeepSparse, run the following:

```bash
deepsparse.server --config_file server/deepsparse-server/server-config-deepsparse.yaml
```

#### 4. Start The App Server

The App Server is built on `FastAPI` and `uvicorn` and orchestrates DeepSparse, Milvus, and MySQL to create a search engine. 

In this example, we have provided a data file in `server/app-server/data/example.csv`. These data are a `CSV` file with `title` and `text` for about 160 articles on various topics. Launch the app server with the following command:

```bash
python3 server/app-server/src/app.py --data_path server/app-server/data/example.csv
```

### Use the Search Engine!

We have provided both a latency testing script and a Jupyter notebook to interact with the server. The Jupyter notebook is self-documenting and is a good starting point to play around with the application.

The latency testing script generates multiple clients to test response time from the server. It provides metrics on both overall query latency as well as metrics on the model serving query latency (the end to end time from the app server querying DeepSparse until a response is returned.) 

You can run with the following command:


## Running on AWS in a VPC with Independent-Scaling
