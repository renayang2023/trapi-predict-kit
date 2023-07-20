<div align="center">

# 🪄 TRAPI Predict Kit

<!--

[![PyPI - Version](https://img.shields.io/pypi/v/trapi-predict-kit.svg?logo=pypi&label=PyPI&logoColor=silver)](https://pypi.org/project/trapi-predict-kit/)
[![PyPI - Python Version](https://img.shields.io/pypi/pyversions/trapi-predict-kit.svg?logo=python&label=Python&logoColor=silver)](https://pypi.org/project/trapi-predict-kit/)
[![license](https://img.shields.io/pypi/l/trapi-predict-kit.svg?color=%2334D058)](https://github.com/MaastrichtU-IDS/trapi-predict-kit/blob/main/LICENSE.txt)
[![code style - black](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black)
-->

[![Test package](https://github.com/MaastrichtU-IDS/trapi-predict-kit/actions/workflows/test.yml/badge.svg)](https://github.com/MaastrichtU-IDS/trapi-predict-kit/actions/workflows/test.yml)
[![Publish package](https://github.com/MaastrichtU-IDS/trapi-predict-kit/actions/workflows/publish.yml/badge.svg)](https://github.com/MaastrichtU-IDS/trapi-predict-kit/actions/workflows/publish.yml)

</div>

A package to help create and deploy Translator Reasoner APIs (TRAPI) from any prediction model exposed as a regular python function.

## 📦️ Installation

This package requires Python >=3.7, simply install it with:

```shell
pip install trapi-predict-kit
```

To also include uvicorn/gunicorn for deployment:

```bash
pip install trapi-predict-kit[web]
```

## 🪄 Usage

### 🔮 Define the prediction endpoint

The `trapi_predict_kit` package provides a decorator `@trapi_predict` to annotate your functions that generate predictions. Predictions generated from functions decorated with `@trapi_predict` can easily be imported in the Translator OpenPredict API, exposed as an API endpoint to get predictions from the web, and queried through the  Translator Reasoner API (TRAPI).

The annotated predict functions are expected to take 2 input  arguments: the input ID (string) and options for the prediction (dictionary). And it should return a dictionary with a list of predicted associated entities hits. Here is an example:

 ```python
from trapi_predict_kit import trapi_predict, PredictOptions, PredictOutput

@trapi_predict(path='/predict',
    name="Get predicted targets for a given entity",
    description="Return the predicted targets for a given entity: drug (DrugBank ID) or disease (OMIM ID), with confidence scores.",
    edges=[
        {
            'subject': 'biolink:Drug',
            'predicate': 'biolink:treats',
            'object': 'biolink:Disease',
        },
        {
            'subject': 'biolink:Disease',
            'predicate': 'biolink:treated_by',
            'object': 'biolink:Drug',
        },
    ],
    nodes={
        "biolink:Disease": {
            "id_prefixes": [
                "OMIM"
            ]
        },
        "biolink:Drug": {
            "id_prefixes": [
                "DRUGBANK"
            ]
        }
    }
)
def get_predictions(
        input_id: str, options: PredictOptions
    ) -> PredictOutput:
    # Add the code the load the model and get predictions here
    predictions = {
        "hits": [
            {
                "id": "DB00001",
                "type": "biolink:Drug",
                "score": 0.12345,
                "label": "Leipirudin",
            }
        ],
        "count": 1,
    }
    return predictions
 ```

### Define the API

You will need to instantiate a `TRAPI` class to deploy a Translator Reasoner API serving a list of prediction functions that have been decorated with `@trapi_predict`. For example:

```python
import logging

from trapi_predict_kit.config import settings
from trapi_predict_kit import TRAPI
# TODO: change to your module name
from my_model.predict import get_predictions

log_level = logging.ERROR
if settings.DEV_MODE:
    log_level = logging.INFO
logging.basicConfig(level=log_level)

predict_endpoints = [ get_predictions ]

openapi_info = {
    "contact": {
        "name": "Firstname Lastname",
        "email": "email@example.com",
        # "x-id": "https://orcid.org/0000-0000-0000-0000",
        "x-role": "responsible developer",
    },
    "license": {
        "name": "MIT license",
        "url": "https://opensource.org/licenses/MIT",
    },
    "termsOfService": 'https://github.com/your-org-or-username/my-model/blob/main/LICENSE.txt',

    "x-translator": {
        "component": 'KP',
        # TODO: update the Translator team to yours
        "team": [ "Clinical Data Provider" ],
        "biolink-version": settings.BIOLINK_VERSION,
        "infores": 'infores:openpredict',
        "externalDocs": {
            "description": "The values for component and team are restricted according to this external JSON schema. See schema and examples at url",
            "url": "https://github.com/NCATSTranslator/translator_extensions/blob/production/x-translator/",
        },
    },
    "x-trapi": {
        "version": settings.TRAPI_VERSION,
        "asyncquery": False,
        "operations": [
            "lookup",
        ],
        "externalDocs": {
            "description": "The values for version are restricted according to the regex in this external JSON schema. See schema and examples at url",
            "url": "https://github.com/NCATSTranslator/translator_extensions/blob/production/x-trapi/",
        },
    }
}

servers = []
if settings.VIRTUAL_HOST:
    servers = [
        {
            "url": f"https://{settings.VIRTUAL_HOST}",
            "description": 'TRAPI ITRB Production Server',
            "x-maturity": 'production'
        },
    ]

app = TRAPI(
    predict_endpoints=predict_endpoints,
    servers=servers,
    info=openapi_info,
    title='My model TRAPI',
    version='1.0.0',
    openapi_version='3.0.1',
    description="""Machine learning models to produce predictions that can be integrated to Translator Reasoner APIs.
\n\nService supported by the [NCATS Translator project](https://ncats.nih.gov/translator/about)""",
    dev_mode=True,
)
```

### Deploy the API

Change `trapi.main` to your module path in the command before running it:

```bash
uvicorn trapi.main:app --port 8808 --reload
```

## 🧑‍💻 Development setup

The final section of the README is for if you want to run the package in development, and get involved by making a code contribution.

### 📥️ Clone

Clone the repository:

```bash
git clone https://github.com/MaastrichtU-IDS/trapi-predict-kit
cd trapi-predict-kit
```

### 🐣 Install dependencies

Install [Hatch](https://hatch.pypa.io), this will automatically handle virtual environments and make sure all dependencies are installed when you run a script in the project:

```bash
pip install --upgrade hatch
```

Install the dependencies in a local virtual environment:

```bash
hatch -v env create
```

To test it locally with python 3.7 use mamba or conda:

```bash
mamba create -n py37 python=3.7
```

### 🧑‍💻 Develop

Run the API for development defined in `tests/dev.py`:

```bash
hatch run api
```

### ☑️ Run tests

Make sure the existing tests still work by running ``pytest``. Note that any pull requests to the fairworkflows repository on github will automatically trigger running of the test suite;

```bash
hatch run test
```

To display all logs when debugging:

```bash
hatch run test -s
```

### 🧹 Code formatting

The code will be automatically formatted when you commit your changes using `pre-commit`. But you can also run the script to format the code yourself:

```
hatch run fmt
```

### ♻️ Reset the environment

In case you are facing issues with dependencies not updating properly you can easily reset the virtual environment with:

```bash
hatch env prune
```

### 🏷️ New release process

The deployment of new releases is done automatically by a GitHub Action workflow when a new release is created on GitHub. To release a new version:

1. Make sure the `PYPI_TOKEN` secret has been defined in the GitHub repository (in Settings > Secrets > Actions). You can get an API token from PyPI at [pypi.org/manage/account](https://pypi.org/manage/account).
2. Increment the `version` number in the `pyproject.toml` file in the root folder of the repository.
3. Create a new release on GitHub, which will automatically trigger the publish workflow, and publish the new release to PyPI.

You can also manually trigger the workflow from the Actions tab in your GitHub repository webpage.
