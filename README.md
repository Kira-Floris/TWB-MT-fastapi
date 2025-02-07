# TWB-MT-fastapi

API for serving machine translation models. 

It can run two types of models:
- [ctranslate2](https://github.com/OpenNMT/CTranslate2) models
- Certain transformer-based models models provided through [huggingface](https://huggingface.co/Helsinki-NLP).
    - OPUS 
    - OPUS-big
    - NLLB (Multilingual)

Model specifications need to go in `config.json`.

## Model installation examples

### Custom model (ctranslate2)

For an English to Turkish model, place the following files under `model/entr`: 

- ctranslator2 model as `model.bin`
- (Optional) BPE subword codes file (e.g. `bpe.en-tr.codes`)
- (Optional) Sentencepiece model (To be implemented)

Add the following configuration under `models` in `config.json`:

```
{
    "src": "en",
    "tgt": "tr",
    "model_type": "ctranslator2",
    "model_path": "entr",  //model directory name under models
    "bpe_file": "bpe.en-tr.codes",
    "load": true,
    "sentence_split": "nltk",
    "pipeline": {
        "lowercase": true,  //make true if you processed your data in lowercase when training your model 
        "tokenize": true,  //make true if you processed your data with moses tokenizer when training your model 
        "bpe": true,  //make true if you use bpe subwording when training your model 
        "translate": true,  
        "recase": true  // true if you want to do recasing on output (capitalizes first letter of sentence)
    }
}
```

### OPUS models (huggingface)

You can serve Helsinki-NLP's OPUS models provided in huggingface, as long as they are one-to-one. Make sure they are listed in https://huggingface.co/Helsinki-NLP and place the language codes exactly as they are. 
For an French to English model, add the following configuration under `models` in `config.json`:

```
{
    "src": "fr",
    "tgt": "en",
    "model_type": "opus",
    "load": true,
    "sentence_split": "nltk" ,
    "pipeline": {
        "lowercase": false,
        "translate": true,
        "recase": true
    }
}
```

Some models have a different architecture like the [English to Turkish model](https://huggingface.co/Helsinki-NLP/opus-mt-tc-big-en-tr). These models needs to have `opus-big` as `model_type`. For example:

```
{
    "src": "en",
    "tgt": "tr",
    "model_type": "opus-big",
    "load": true,
    "sentence_split": "nltk", 
    "pipeline": {
        "translate": true
    }
}
```

### NLLB model

[NLLB (No Language Left Behind)](https://github.com/facebookresearch/fairseq/tree/nllb) is a multilingal MT model developed by Meta AI that supports [200 languages](https://github.com/facebookresearch/flores/blob/main/flores200/README.md#languages-in-flores-200). Model checkpoints of various sizes are [supported through huggingface](https://huggingface.co/docs/transformers/model_doc/nllb) and can be loaded in the API by specifying the checkpoint id and language pairs to be activated in the API configuration. 

Example configuration supporting bidirectional English-Kanuri, English-French, English-Fulfulde, English-Hausa:

```
{
  "model_type": "nllb",
  "nllb_checkpoint_id": "nllb-200-distilled-600M", 
  "multilingual": true,
  "load": true,
  "sentence_split": "nltk",
  "supported_pairs": ["en-kr", "en-fr", "en-ff", "en-ha"],
  "pipeline": {
      "translate": true
  }
}
```

Depending on your server architecture, you can choose `nllb_checkpoint_id` from `nllb-200-distilled-1.3B`, `nllb-200-distilled-600M` or `nllb-200-3.3B`.

By convention, we use languages in two lettered ISO codes in the configuration file. `app/constants.py` contains the mappings from these codes into the codes used by the NLLB model. This mapping is currently incomplete, so, if you need to add a new language and want to use a language ID other than the one used by NLLB model, you add the mapping into this dictionary. If you prefer, you can use the NLLB id directly in the configuration file as well. The complete list of 200 languages and their respective codes can be viewed through [Flores200 README file](https://github.com/facebookresearch/flores/blob/main/flores200/README.md#languages-in-flores-200). 

```
NLLB_LANGS_DICT = {'en': 'eng_Latn', 'tr': 'tur_Latn', 'fr': 'fra_Latn',
                   'kr': 'knc_Latn', 'ha': 'hau_Latn', 'ff': 'fuv_Latn'}
```

## Build and run

Set the environment variables:
```
MT_API_CONFIG=config.json
MODELS_ROOT=../translation-models
MT_API_DEVICE=cpu #or "gpu"
```

```
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8001
```

You can also run `run_local.sh` directly. 

## Build and run with docker-compose (recommended)

```
docker-compose build
docker-compose up
```

### To use GPU on docker

Do the following edits on docker-compose file
1. Remove comment on `runtime: nvidia` line
2. Under environment, set `MT_API_DEVICE=gpu`
3. Build and run.

## Example calls

### Simple translation

Endpoint for translating a single phrase. 

#### cURL

```
curl --location --request POST 'http://127.0.0.1:8001/api/v1/translate' \
--header 'Content-Type: application/json' \
--data-raw '{"src":"fr", "tgt":"en", "text":"c'\''est un test"}'
```

#### Python

```
import httpx
translate_service_url = "http://127.0.0.1:8001/api/v1/translate"
json_data = {'src':'fr', 'tgt':'en', 'text':"c'est un test."}
r = httpx.post(translate_service_url, json=json_data)
response = r.json()
print("Translation:", response['translation'])
```

### Batch translation

Endpoint for translating a list of sentences.

#### cURL

```
curl --location --request POST 'http://127.0.0.1:8001/api/v1/translate/batch' \
--header 'Content-Type: application/json' \
--data-raw '{"src":"en", "tgt":"fr", "texts":["hello twb", "this is another sentence"]}'
```

#### Python

```
import httpx
translate_service_url = "http://127.0.0.1:8001/api/v1/translate/batch"
json_data = {'src':'fr', 'tgt':'en', 'texts':["hello twb", "this is another sentence"]}
r = httpx.post(translate_service_url, json=json_data)
response = r.json()
print("Translation:", response['translation'])
```

### Retrieve languages

Retrieves a the list of supported languages and model pairs.

#### cURL

```
curl 'http://127.0.0.1:8001/api/v1/translate/'
```

#### Python
```
import httpx
translate_service_url = "http://127.0.0.1:8001/api/v1/translate/"
r = httpx.get(translate_url)
response = r.json()
print(response)
```

#### Response description

```
{
  "languages": {
    "ar": "Levantine Arabic",
    "en": "English",
    "fr": "French",
    "swc": "Congolese Swahili",
    "ti": "Tigrinya"
  }, "models":{"fr":{"en":["fr_en"],"swc":["fr_swc"]},"swc":{"fr":["swc_fr"]},"ti":{"en":["ti_en"]},"en":{"ti":["en_ti"]},"ar":{"en":["ar_en", "ar_en_domainspecific"]}}
}
```

- `languages`: All language codes used in the system and their respective language names.
- `models`: Three level dictionary listing the available models with the structure:
```
    <source-language>
        ↳ <target-language>
            ↳ List of <model-id>s associated with the language pair
```

Note: The third level model list is for seeing different versions of the model. There is always a default model in the language pair with id `<src>_<tgt>` and there could be alternative models with model id `<src>_<tgt>_<alt-tag>`. 

For example in the setup above, there are two alternative models in the Arabic-English direction: A default model `ar_en` and a domain specific `ar_en_domainspecific` model where domainspecific is the alternative model id. For the rest of the language pairs there is only one default model. 

