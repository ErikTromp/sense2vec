<a href="https://explosion.ai"><img src="https://explosion.ai/assets/img/logo.svg" width="125" height="125" align="right" /></a>

# sense2vec: Use NLP to go beyond vanilla word2vec

sense2vec [Trask et. al](https://arxiv.org/abs/1511.06388), 2015) is a nice
twist on [word2vec](https://en.wikipedia.org/wiki/Word2vec) that lets you learn
more interesting, detailed and context-sensitive word vectors. For an
interactive example of the technology, see our
[sense2vec demo](https://demos.explosion.ai/sense2vec) that lets you explore
semantic similarities across all Reddit comments of 2015. This library is a
simple Python implementation for loading and querying sense2vec models.

🦆 **Version 1.0 out now!**
[Read the release notes here.](https://github.com/explosion/sense2vec/releases/)

[![Azure Pipelines](https://img.shields.io/azure-devops/build/explosion-ai/public/12/master.svg?logo=azure-pipelines&style=flat-square&label=build)](https://dev.azure.com/explosion-ai/public/_build?definitionId=12)
[![Current Release Version](https://img.shields.io/github/v/release/explosion/sense2vec.svg?style=flat-square&include_prereleases&logo=github)](https://github.com/explosion/sense2vec/releases)
[![pypi Version](https://img.shields.io/pypi/v/sense2vec.svg?style=flat-square&logo=pypi&logoColor=white)](https://pypi.org/project/sense2vec/)
[![Code style: black](https://img.shields.io/badge/code%20style-black-000000.svg?style=flat-square)](https://github.com/ambv/black)

## Usage Examples

### Standalone usage

```python
from sense2vec import Sense2Vec

s2v = Sense2Vec().from_disk("/path/to/reddit_vectors-1.1.0")
query = "natural_language_processing|NOUN"
assert query in s2v
vector = s2v[query]
freq = s2v.get_freq(query)
most_similar = s2v.most_similar(query, n=3)
# [('machine_learning|NOUN', 0.8986967),
#  ('computer_vision|NOUN', 0.8636297),
#  ('deep_learning|NOUN', 0.8573361)]
```

### Usage as a spaCy pipeline component

```python
import spacy
from sense2vec import Sense2VecComponent

nlp = spacy.load("en_core_web_sm")
s2v = Sense2VecComponent(nlp.vocab).from_disk("/path/to/reddit_vectors-1.1.0")
nlp.add_pipe(s2v)

doc = nlp("A sentence about natural language processing.")
assert doc[3:6].text == "natural language processing"
freq = doc[3:6]._.s2v_freq
vector = doc[3:6]._.s2v_vec
most_similar = doc[3:6]._.s2v_most_similar(3)
# [(('machine learning', 'NOUN'), 0.8986967),
#  (('computer vision', 'NOUN'), 0.8636297),
#  (('deep learning', 'NOUN'), 0.8573361)]
```

## Installation & Setup

sense2vec releases are available on pip:

```bash
pip install sense2vec
```

The Reddit vectors model is attached to the
[latest release](https://github.com/explosion/sense2vec/releases). To load it
in, download the `.tar.gz` archive, unpack it and point `from_disk` to the
extracted data directory:

```python
from sense2vec import Sense2Vec
s2v = Sense2Vec().from_disk("/path/to/reddit_vectors-1.1.0")
```

## Usage

## Usage with spaCy v2.x

The easiest way to use the library and vectors is to plug it into your spaCy
pipeline. Note that `sense2vec` doesn't depend on spaCy, so you'll have to
install it separately and download the English model.

```bash
pip install -U spacy
python -m spacy download en_core_web_sm
```

The `sense2vec` package exposes a `Sense2VecComponent`, which can be initialised
with the shared vocab and added to your spaCy pipeline as a
[custom pipeline component](https://spacy.io/usage/processing-pipelines#custom-components).
By default, components are added to the _end of the pipeline_, which is the
recommended position for this component, since it needs access to the dependency
parse and, if available, named entities.

```python
import spacy
from sense2vec import Sense2VecComponent

nlp = spacy.load("en_core_web_sm")
s2v = Sense2VecComponent(nlp.vocab).from_disk("/path/to/reddit_vectors-1.1.0")
nlp.add_pipe(s2v)
```

The pipeline component will **merge noun phrases and entities** according to the
same schema used when training the sense2vec models (e.g. noun chunks without
determiners like "the"). This ensures that you'll be able to retrieve meaningful
vectors for phrases in your text. The component will also add serveral
[extension attributes and methods](https://spacy.io/usage/processing-pipelines#custom-components-attributes)
to spaCy's `Token` and `Span` objects that let you retrieve vectors and
frequencies, as well as most similar terms.

```python
doc = nlp("A sentence about natural language processing.")
assert doc[3:6].text == "natural language processing"
freq = doc[3:6]._.s2v_freq
vector = doc[3:6]._.s2v_vec
most_similar = doc[3:6]._.s2v_most_similar(3)
```

For entities, the entity labels are used as the "sense" (instead of the token's
part-of-speech tag):

```python
doc = nlp("A sentence about Facebook and Google.")
for ent in doc.ents:
    assert ent._.in_s2v
    most_similar = ent._.s2v_most_similar(3)
```

### Available attributes

The following extension attributes are exposed on the `Doc` object via the `._`
property:

| Name          | Attribute Type | Type | Description                                                                         |
| ------------- | -------------- | ---- | ----------------------------------------------------------------------------------- |
| `s2v_phrases` | property       | list | All sense2vec-compatible phrases in the given `Doc` (noun phrases, named entities). |

The following attributes are available via the `._` property of `Token` and
`Span` objects – for example `token._.in_s2v`:

| Name               | Attribute Type | Type               | Description                                                                        |
| ------------------ | -------------- | ------------------ | ---------------------------------------------------------------------------------- |
| `in_s2v`           | property       | bool               | Whether a key exists in the vector map.                                            |
| `s2v_key`          | property       | unicode            | The sense2vec key of the given object, e.g. `"duck|NOUN"`.                         |
| `s2v_vec`          | property       | `ndarray[float32]` | The vector of the given key.                                                       |
| `s2v_freq`         | property       | int                | The frequency of the given key.                                                    |
| `s2v_other_senses` | property       | list               | Available other senses, e.g. `"duck|VERB"` for `"duck|NOUN"`.                      |
| `s2v_most_similar` | method         | list               | Get the `n` most similar terms. Returns a list of `((word, sense), score)` tuples. |

> ⚠️ **A note on span attributes:** Under the hood, entities in `doc.ents` are
> `Span` objects. This is why the pipeline component also adds attributes and
> methods to spans and not just tokens. However, it's not recommended to use the
> sense2vec attributes on arbitrary slices of the document, since the model
> likely won't have a key for the respective text. `Span` objects also don't
> have a part-of-speech tag, so if no entity label is present, the "sense"
> defaults to the root's part-of-speech tag.

### Standalone usage

You can also use the underlying `Sense2Vec` class directly and load in the
vectors using the `from_disk` method. See below for the available API methods.

```python
from sense2vec import Sense2Vec
s2v = Sense2Vec().from_disk("/path/to/reddit_vectors-1.1.0")
most_similar = s2v.most_similar("natural_language_processing|NOUN", n=10)
```

> ⚠️ **Important note:** To look up entries in the vectors table, the keys need
> to follow the scheme of `phrase_text|SENSE` (note the `_` instead of spaces
> and the `|` before the tag or label) – for example, `machine_learning|NOUN`.
> Also note that the underlying vector table is case-sensitive.

## 🎛 API

### <kbd>method</kbd> `Sense2Vec.__init__`

Initialize the `Sense2Vec` object.

| Argument    | Type                        | Description                                                                                                 |
| ----------- | --------------------------- | ----------------------------------------------------------------------------------------------------------- |
| `shape`     | tuple                       | The vector shape. Defaults to `(1000, 128)`.                                                                |
| `strings`   | `spacy.strings.StringStore` | Optional string store. Will be created if it doesn't exist.                                                 |
| `make_key`  | callable                    | Optional custom function that takes a word and sense string and creates the key (e.g. `"some_word|sense"`). |
| `split_key` | callable                    | Optional custom function that takes a key and returns the word and sense (e.g. `("some word", "sense")`).   |
| `senses`    | list                        | Optional list of all available senses. Used in methods that generate the best sense or other senses.        |
| **RETURNS** | `Sense2Vec`                 | The newly constructed object.                                                                               |

```python
s2v = Sense2Vec(shape=(300, 128), senses=["VERB", "NOUN"])
```

### <kbd>method</kbd> `Sense2Vec.__len__`

The number of rows in the vectors table.

| Argument    | Type | Description                              |
| ----------- | ---- | ---------------------------------------- |
| **RETURNS** | int  | The number of rows in the vectors table. |

```python
s2v = Sense2Vec(shape=(300, 128))
assert len(s2v) == 300
```

### <kbd>method</kbd> `Sense2Vec.__contains__`

Check if a key is in the vectors table.

| Argument    | Type          | Description                      |
| ----------- | ------------- | -------------------------------- |
| `key`       | unicode / int | The key to look up.              |
| **RETURNS** | bool          | Whether the key is in the table. |

```python
s2v = Sense2Vec(shape=(10, 4))
s2v.add("avocado|NOUN", numpy.asarray([4, 2, 2, 2], dtype=numpy.float32))
assert "avocado|NOUN" in s2v
assert "avocado|VERB" not in s2v
```

### <kbd>method</kbd> `Sense2Vec.__getitem__`

Retrieve a vector for a given key.

| Argument    | Type            | Description         |
| ----------- | --------------- | ------------------- |
| `key`       | unicode / int   | The key to look up. |
| **RETURNS** | `numpy.ndarray` | The vector.         |

```python
vec = s2v["avocado|NOUN"]
```

### <kbd>method</kbd> `Sense2Vec.__setitem__`

Set a vector for a given key. Will raise an error if the key doesn't exist. To
add a new entry, use `Sense2Vec.add`.

| Argument | Type            | Description        |
| -------- | --------------- | ------------------ |
| `key`    | unicode / int   | The key.           |
| `vector` | `numpy.ndarray` | The vector to set. |

```python
vec = s2v["avocado|NOUN"]
s2v["avacado|NOUN"] = vec
```

### <kbd>method</kbd> `Sense2Vec.add`

Add a new vector to the table.

| Argument | Type            | Description                                                  |
| -------- | --------------- | ------------------------------------------------------------ |
| `key`    | unicode / int   | The key to add.                                              |
| `vector` | `numpy.ndarray` | The vector to add.                                           |
| `freq`   | int             | Optional frequency count. Used to find best matching senses. |

```python
vec = s2v["avocado|NOUN"]
s2v.add("🥑|NOUN", vec, 1234)
```

### <kbd>method</kbd> `Sense2Vec.get_freq`

Get the frequency count for a given key.

| Argument    | Type          | Description                                       |
| ----------- | ------------- | ------------------------------------------------- |
| `key`       | unicode / int | The key to look up.                               |
| `default`   | -             | Default value to return if no frequency is found. |
| **RETURNS** | int           | The frequency count.                              |

```python
vec = s2v["avocado|NOUN"]
s2v.add("🥑|NOUN", vec, 1234)
assert s2v.get_freq("🥑|NOUN") == 1234
```

### <kbd>method</kbd> `Sense2Vec.set_freq`

Set a frequency count for a given key.

| Argument | Type          | Description                   |
| -------- | ------------- | ----------------------------- |
| `key`    | unicode / int | The key to set the count for. |
| `freq`   | int           | The frequency count.          |

```python
s2v.set_freq("avocado|NOUN", 104294)
```

### <kbd>method</kbd> `Sense2Vec.__iter__`, `Sense2Vec.items`

Iterate over the entries in the vectors table.

| Argument   | Type  | Description                               |
| ---------- | ----- | ----------------------------------------- |
| **YIELDS** | tuple | String key and vector pairs in the table. |

```python
for key, vec in s2v:
    print(key, vec)

for key, vec in s2v.items():
    print(key, vec)
```

### <kbd>method</kbd> `Sense2Vec.keys`

Iterate over the keys in the table.

| Argument   | Type    | Description                   |
| ---------- | ------- | ----------------------------- |
| **YIELDS** | unicode | The string keys in the table. |

```python
all_keys = list(s2v.keys())
```

### <kbd>method</kbd> `Sense2Vec.values`

Iterate over the vectors in the table.

| Argument   | Type            | Description               |
| ---------- | --------------- | ------------------------- |
| **YIELDS** | `numpy.ndarray` | The vectors in the table. |

```python
all_vecs = list(s2v.values())
```

### <kbd>property</kbd> `Sense2Vec.senses`

The available senses in the table, e.g. `"NOUN"` or `"VERB"` (added at
initialization).

| Argument    | Type | Description           |
| ----------- | ---- | --------------------- |
| **RETURNS** | list | The available senses. |

```python
s2v = Sense2Vec(senses=["VERB", "NOUN"])
assert "VERB" in s2v.senses
```

### <kbd>method</kbd> `Sense2Vec.most_similar`

Get the most similar entries in the table.

| Argument     | Type                      | Description                                             |
| ------------ | ------------------------- | ------------------------------------------------------- |
| `keys`       | unicode / int / iterable  | The string or integer key(s) to compare to.             |
| `n`          | int                       | The number of similar keys to return. Defaults to `10`. |
| `batch_size` | int                       | The batch size to use. Defaults to `16`.                |
| **RETURNS**  | list                      | The `(key, score)` tuples of the most similar vectors.  |

```python
most_similar = s2v.most_similar("natural_language_processing|NOUN", n=3)
# [('machine_learning|NOUN', 0.8986967),
#  ('computer_vision|NOUN', 0.8636297),
#  ('deep_learning|NOUN', 0.8573361)]
```

### <kbd>method</kbd> `Sense2Vec.get_other_senses`

Find other entries for the same word with a different sense, e.g. `"duck|VERB"`
for `"duck|NOUN"`.

| Argument      | Type          | Description                                                       |
| ------------- | ------------- | ----------------------------------------------------------------- |
| `key`         | unicode / int | The key to check.                                                 |
| `ignore_case` | bool          | Check for uppercase, lowercase and titlecase. Defaults to `True`. |
| **RETURNS**   | list          | The string keys of other entries with different senses.           |

```python
other_senses = s2v.get_other_senses("duck|NOUN")
# ['duck|VERB', 'Duck|ORG', 'Duck|VERB', 'Duck|PERSON', 'Duck|ADJ']
```

### <kbd>method</kbd> `Sense2Vec.get_best_sense`

Find the best-matching sense for a given word based on the available senses and
frequency counts. Returns `None` if no match is found.

| Argument      | Type    | Description                                                       |
| ------------- | ------- | ----------------------------------------------------------------- |
| `word`        | unicode | The word to check.                                                |
| `ignore_case` | bool    | Check for uppercase, lowercase and titlecase. Defaults to `True`. |
| **RETURNS**   | unicode | The best-matching key or None.                                    |

```python
assert s2v.get_best_sense("duck") == "duck|NOUN"
```

### <kbd>method</kbd> `Sense2Vec.to_bytes`

Serialize a `Sense2Vec` object to a bytestring.

| Argument    | Type  | Description                               |
| ----------- | ----- | ----------------------------------------- |
| `exclude`   | list  | Names of serialization fields to exclude. |
| **RETURNS** | bytes | The serialized `Sense2Vec` object.        |

```python
s2v_bytes = s2v.to_bytes()
```

### <kbd>method</kbd> `Sense2Vec.from_bytes`

Load a `Sense2Vec` object from a bytestring.

| Argument     | Type        | Description                               |
| ------------ | ----------- | ----------------------------------------- |
| `bytes_data` | bytes       | The data to load.                         |
| `exclude`    | list        | Names of serialization fields to exclude. |
| **RETURNS**  | `Sense2Vec` | The loaded object.                        |

```python
s2v_bytes = s2v.to_bytes()
new_s2v = Sense2Vec().from_bytes(s2v_bytes)
```

### <kbd>method</kbd> `Sense2Vec.to_disk`

Serialize a `Sense2Vec` object to a directory.

| Argument  | Type             | Description                               |
| --------- | ---------------- | ----------------------------------------- |
| `path`    | unicode / `Path` | The path.                                 |
| `exclude` | list             | Names of serialization fields to exclude. |

```python
s2v.to_disk("/path/to/sense2vec")
```

### <kbd>method</kbd> `Sense2Vec.from_disk`

Load a `Sense2Vec` object from a directory.

| Argument    | Type             | Description                               |
| ----------- | ---------------- | ----------------------------------------- |
| `path`      | unicode / `Path` | The path. to load from                    |
| `exclude`   | list             | Names of serialization fields to exclude. |
| **RETURNS** | `Sense2Vec`      | The loaded object.                        |

```python
s2v.to_disk("/path/to/sense2vec")
new_s2v = Sense2Vec().from_disk("/path/to/sense2vec")
```

## Pre-trained vectors

The pre-trained Reddit vectors support the following "senses", either
part-of-speech tags or entity labels. For more details, see spaCy's
[annotation scheme overview](https://spacy.io/api/annotation).

| Tag     | Description               | Examples                             |
| ------- | ------------------------- | ------------------------------------ |
| `ADJ`   | adjective                 | big, old, green                      |
| `ADP`   | adposition                | in, to, during                       |
| `ADV`   | adverb                    | very, tomorrow, down, where          |
| `AUX`   | auxiliary                 | is, has (done), will (do)            |
| `CONJ`  | conjunction               | and, or, but                         |
| `DET`   | determiner                | a, an, the                           |
| `INTJ`  | interjection              | psst, ouch, bravo, hello             |
| `NOUN`  | noun                      | girl, cat, tree, air, beauty         |
| `NUM`   | numeral                   | 1, 2017, one, seventy-seven, MMXIV   |
| `PART`  | particle                  | 's, not                              |
| `PRON`  | pronoun                   | I, you, he, she, myself, somebody    |
| `PROPN` | proper noun               | Mary, John, London, NATO, HBO        |
| `PUNCT` | punctuation               | , ? ( )                              |
| `SCONJ` | subordinating conjunction | if, while, that                      |
| `SYM`   | symbol                    | \$, %, =, :), 😝                     |
| `VERB`  | verb                      | run, runs, running, eat, ate, eating |

| Entity Label  | Description                                          |
| ------------- | ---------------------------------------------------- |
| `PERSON`      | People, including fictional.                         |
| `NORP`        | Nationalities or religious or political groups.      |
| `FACILITY`    | Buildings, airports, highways, bridges, etc.         |
| `ORG`         | Companies, agencies, institutions, etc.              |
| `GPE`         | Countries, cities, states.                           |
| `LOC`         | Non-GPE locations, mountain ranges, bodies of water. |
| `PRODUCT`     | Objects, vehicles, foods, etc. (Not services.)       |
| `EVENT`       | Named hurricanes, battles, wars, sports events, etc. |
| `WORK_OF_ART` | Titles of books, songs, etc.                         |
| `LANGUAGE`    | Any named language.                                  |
