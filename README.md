# Manifest
How to make prompt programming with FMs a little easier.

## Install
Download the code:
```bash
git clone git@github.com:HazyResearch/manifest.git
cd manifest
```

Install:
```bash
pip install poetry
poetry install --no-dev
```

Dev Install:
```bash
pip install poetry
make dev
```

## Getting Started
Running is simple to get started. If using OpenAI, set `export OPENAI_API_KEY=<OPENAIKEY>` then run

```python
from manifest import Manifest

# Start a manifest session
manifest = Manifest(
    client_name = "openai",
)
manifest.run("Why is the grass green?")
```

We also support AI21, OPT models, and HuggingFace models (see [below](#huggingface-models)).

Caching by default is turned off, but to cache results, run

```python
from manifest import Manifest

# Start a manifest session
manifest = Manifest(
    client_name = "openai",
    cache_name = "sqlite",
    cache_connection = "mycache.sqlite",
)
manifest.run("Why is the grass green?")
```

We also support Redis backend.

## Manifest Components
Manifest is meant to be a very light weight package to help with prompt iteration. Three key design decisions are

* Prompt are functional -- they can take an input example and dynamically change
* All models are behind API calls (e.g., OpenAI)
* Everything can cached for reuse to both save credits and to explore past results

### Prompts
A Manifest prompt is a function that accepts a single input to generate a string prompt to send to a model.

```python
from manifest import Prompt
prompt = Prompt(lambda x: "Hello, my name is {x}")
print(prompt("Laurel"))
>>> "Hello, my name is Laurel"
```

We also let you use static strings
```python
prompt = Prompt("Hello, my name is static")
print(prompt())
>>> "Hello, my name is static"
```

### Sessions

Each Manifest run is a session that connects to a model endpoint and backend database to record prompt queries. To start a Manifest session for OpenAI, make sure you run
```bash
export OPENAI_API_KEY=<OPENAIKEY>
```
so we can access OpenAI.

Then run:
```python
from manifest import Manifest

manifest = Manifest(
    client_name = "openai",
    cache_name = "sqlite",
    cache_connection = "sqlite.cache"
)
```
This will start a session with OpenAI and save all results to a local file called `sqlite.cache`.

We also support a Redis backend. If you have a Redis database running on port 6379, run
```python
manifest = Manifest(
    client_name = "openai",
    cache_name = "redis",
    cache_connection = "localhost:6379"
)
```
As a hint, if you want to get Redis running, see the `docker run` command below under development.

We will explain [below](#huggingface-models) how to use Manifest for a locally hosted HuggingFace model.

Once you have a session open, you can write and develop prompts.

```python
prompt = Prompt(lambda x: "Hello, my name is {x}")
result = manifest.run(prompt, "Laurel")
```

You can also run over multiple examples.
```python
results = manifest.batch_run(prompt, ["Laurel", "Avanika"])
```

If something doesn't go right, you can also ask to get a raw manifest Response.
```python
result_object = manifest.batch_run(prompt, ["Laurel", "Avanika"], return_response=True)
print(result_object.get_request())
print(result_object.is_cached())
print(result_object.get_response())
```

By default, we do not truncate results based on a stop token. You can change this by either passing a new stop token to a Manifest session or to a `run` or `batch_run`. If you set the stop token to `""`, we will not truncate the model output.
```python
result = manifest.run(prompt, "Laurel", stop_token="and")
```

If you want to change default parameters to a model, we pass those as `kwargs` to the client.
```python
result = manifest.run(prompt, "Laurel", max_tokens=50)
```

### Huggingface Models
To use a HuggingFace generative model, in `manifest/api` we have a Falsk application that hosts the models for you.

In a separate terminal or Tmux/Screen session, run
```python
python3 manifest/api/app.py --model_type huggingface --model_name EleutherAI/gpt-j-6B --device 0
```
You will see the Flask session start and output a URL `http://127.0.0.1:5000`. Pass this in to Manifest. If you want to use a different port, set the `FLASK_PORT` environment variable.

```python
manifest = Manifest(
    client_name = "huggingface",
    client_connection = "http://127.0.0.1:5000",
)
```

If you have a custom model you trained, pass the model path to `--model_name`.

## Development
Before submitting a PR, run
```bash
export REDIS_PORT="6380"  # or whatever PORT local redis is running for those tests
cd <REDIS_PATH>
docker run -d -p 127.0.0.1:${REDIS_PORT}:6380 -v `pwd`:`pwd` -w `pwd` --name manifest_redis_test redis
make test
```

To use our development Redis database, email [Laurel](lorr1@cs.stanford.edu). If you have access to our GCP account, in a separate terminal, run
```bash
gcloud compute ssh "manifest-connect" --zone "europe-west4-a" --project "hai-gcp-head-models" -- -N -L 6379:10.152.93.107:6379
```

Then if you issue
```bash
redis-cli ping
```
You should see a `PONG` response from our database.
