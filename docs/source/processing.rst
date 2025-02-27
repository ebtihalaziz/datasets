Processing data in a Dataset
==============================================================

🤗Datasets provides many methods to modify a Dataset, be it to reorder, split or shuffle the dataset or to apply data processing functions or evaluation functions to its elements.

We'll start by presenting the methods which change the order or number of elements before presenting methods which access and can change the content of the elements themselves.

As always, let's start by loading a small dataset for our demonstrations:

.. code-block::

    >>> from datasets import load_dataset
    >>> dataset = load_dataset('glue', 'mrpc', split='train')

.. note::

    **No in-place policy** All the methods in this chapter return a new :class:`datasets.Dataset`. No modification is done in-place and it's the thus responsibility of the user to decide to override the previous dataset with the newly returned one.

.. note::

    **Caching policy** All the methods in this chapter store the updated dataset in a cache file indexed by a hash of current state and all the argument used to call the method.

    A subsequent call to any of the methods detailed here (like :func:`datasets.Dataset.sort`, :func:`datasets.Dataset.map`, etc) will thus **reuse the cached file instead of recomputing the operation** (even in another python session).

    This usually makes it very efficient to process data with 🤗Datasets.

    If the disk space is critical, these methods can be called with arguments to avoid this behavior (see the last section), or the cache files can be cleaned using the method :func:`datasets.Dataset.cleanup_cache_files`.


Selecting, sorting, shuffling, splitting rows
--------------------------------------------------

Several methods are provided to reorder rows and/or split the dataset:

- sorting the dataset according to a column (:func:`datasets.Dataset.sort`)
- shuffling the dataset (:func:`datasets.Dataset.shuffle`)
- filtering rows either according to a list of indices (:func:`datasets.Dataset.select`) or with a filter function returning true for the rows to keep (:func:`datasets.Dataset.filter`),
- splitting the dataset in a (potentially shuffled) train and a test split (:func:`datasets.Dataset.train_test_split`),
- splitting the dataset in a deterministic list of shards (:func:`datasets.Dataset.shard`),
- concatenate datasets that have the same column types (:func:`datasets.concatenate_datasets`).

These methods have quite simple signature and should be for the most part self-explanatory.

Let's see them in action:

Sorting the dataset according to a column: ``sort``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The provided column has to be NumPy compatible (typically a column containing numerical values).

.. code-block::

    >>> dataset['label'][:10]
    [1, 0, 1, 0, 1, 1, 0, 1, 0, 0]
    >>> sorted_dataset = dataset.sort('label')
    >>> sorted_dataset['label'][:10]
    [0, 0, 0, 0, 0, 0, 0, 0, 0, 0]
    >>> sorted_dataset['label'][-10:]
    [1, 1, 1, 1, 1, 1, 1, 1, 1, 1]

Shuffling the dataset: ``shuffle``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block::

    >>> shuffled_dataset = sorted_dataset.shuffle(seed=42)
    >>> shuffled_dataset['label'][:10]
    [1, 1, 1, 0, 1, 1, 1, 1, 1, 0]

You can also provide a :obj:`numpy.random.Generator` to :func:`datasets.Dataset.shuffle` to control more finely the algorithm used to shuffle the dataset.

Filtering rows: ``select`` and ``filter``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

You can filter rows according to a list of indices (:func:`datasets.Dataset.select`) or with a filter function returning true for the rows to keep (:func:`datasets.Dataset.filter`):

.. code-block::

    >>> small_dataset = dataset.select([0, 10, 20, 30, 40, 50])
    >>> len(small_dataset)
    6

    >>> start_with_ar = dataset.filter(lambda example: example['sentence1'].startswith('Ar'))
    >>> len(start_with_ar)
    6
    >>> start_with_ar['sentence1']
    ['Around 0335 GMT , Tab shares were up 19 cents , or 4.4 % , at A $ 4.56 , having earlier set a record high of A $ 4.57 .',
     'Arison said Mann may have been one of the pioneers of the world music movement and he had a deep love of Brazilian music .',
     'Arts helped coach the youth on an eighth-grade football team at Lombardi Middle School in Green Bay .',
     'Around 9 : 00 a.m. EDT ( 1300 GMT ) , the euro was at $ 1.1566 against the dollar , up 0.07 percent on the day .',
     "Arguing that the case was an isolated example , Canada has threatened a trade backlash if Tokyo 's ban is not justified on scientific grounds .", 
     'Artists are worried the plan would harm those who need help most - performers who have a difficult time lining up shows .'
    ]

:func:`datasets.Dataset.filter` expects a function which can accept a single example of the dataset, i.e. the python dictionary returned by :obj:`dataset[i]` and returns a boolean value. It's also possible to use the indice of each example in the function by setting :obj:`with_indices=True` in :func:`datasets.Dataset.filter`. In this case, the signature of the function given to :func:`datasets.Dataset.filter` should be :obj:`function(example: dict, indice: int) -> bool`:

.. code-block::

    >>> even_dataset = dataset.filter(lambda example, indice: indice % 2 == 0, with_indices=True)
    >>> len(even_dataset)
    1834
    >>> len(dataset) / 2
    1834.0

Splitting the dataset in train and test split: ``train_test_split``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This method is adapted from scikit-learn celebrated :obj:`train_test_split` method with the omission of the stratified options.

You can select the test and train sizes as relative proportions or absolute number of samples.

The splits will be **shuffled by default** using the above described :func:`datasets.Dataset.shuffle` method. You can deactivate this behavior by setting :obj:`shuffle=False` in the arguments of :func:`datasets.Dataset.train_test_split`.

The two splits are returned as a dictionary of :class:`datasets.Dataset`.

.. code-block::

    >>> dataset.train_test_split(test_size=0.1)
    {'train': Dataset(schema: {'sentence1': 'string', 'sentence2': 'string', 'label': 'int64', 'idx': 'int32'}, num_rows: 3301),
     'test': Dataset(schema: {'sentence1': 'string', 'sentence2': 'string', 'label': 'int64', 'idx': 'int32'}, num_rows: 367)}
    >>> 0.1 * len(dataset)
    366.8

We can see that the test split is 10% of the original dataset.

The :func:`datasets.Dataset.train_test_split` has many ways to select the relative sizes of the train and test split so we refer the reader to the package reference of :func:`datasets.Dataset.train_test_split` for all the details.

Sharding the dataset: ``shard``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Eventually, it's possible to "shard" the dataset, i.e. divide it in a deterministic list of dataset of (almost) the same size.

The :func:`datasets.Dataset.shard` takes as arguments the total number of shards (:obj:`num_shards`) and the index of the currently requested shard (:obj:`index`)  and return a :class:`datasets.Dataset` instance constituted by the requested shard.

This method can be used to slice a very large dataset in a predefined number of chunks.


Renaming, removing, casting and flattening columns
--------------------------------------------------

Renaming a column: ``rename_column``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This method renames a column in the dataset, and moves the features associated to the original column under the new column name. This operation will fail if the new column name already exists.

:func:`datasets.Dataset.rename_column` takes the name of the original column and the new name as arguments.

.. code-block::

    >>> dataset = dataset.rename_column("sentence1", "sentenceA")
    >>> dataset = dataset.rename_column("sentence2", "sentenceB")
    >>> dataset
    Dataset({
        features: ['sentenceA', 'sentenceB', 'label', 'idx'],
        num_rows: 3668
    })


Removing one or several columns: ``remove_columns``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

It allows to remove one or several column(s) in the dataset and the features associated to them.

You can also remove a column using :func:`Dataset.map` with `remove_columns` but the present method
doesn't copy the data to a new dataset object and is thus faster.

:func:`datasets.Dataset.remove_columns` takes the names of the column to remove as argument.
You can provide one single column name or a list of column names.

.. code-block::

    >>> dataset = dataset.remove_columns("label")
    >>> dataset
    Dataset({
        features: ['sentence1', 'sentence2', 'idx'],
        num_rows: 3668
    })
    >>> dataset = dataset.remove_columns(['sentence1', 'sentence2'])
    >>> dataset
    Dataset({
        features: ['idx'],
        num_rows: 3668
    })

Casting the dataset to a new set of features types: ``cast``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This method is used to cast the dataset to a new set of features.
You can change the feature type of one or several columns.

For the dataset casting to work, the original features type and the new feature types must be compatible for casting one to the other.
For example you can cast a column with the feature type ``Value("int32")`` to ``Value("bool")`` only if it only contains ones and zeros.
In general, you can only cast a column to a new type if pyarrow allows to cast between the underlying pyarrow data types.

:func:`datasets.Dataset.cast` takes the new :obj:`datasets.Features` definition as argument.

In this example, we change the :obj:`datasets.ClassLabel` label names, and we also change the ``idx`` from ``int32`` to ``int64``:

.. code-block::

    >>> dataset.features
    {'sentence1': Value(dtype='string', id=None),
    'sentence2': Value(dtype='string', id=None),
    'label': ClassLabel(num_classes=2, names=['not_equivalent', 'equivalent'], names_file=None, id=None),
    'idx': Value(dtype='int32', id=None)}
    >>> from datasets import ClassLabel, Value
    >>> new_features = dataset.features.copy()
    >>> new_features["label"] = ClassLabel(names=['negative', 'positive'])
    >>> new_features["idx"] = Value('int64')
    >>> dataset = dataset.cast(new_features)
    >>> dataset.features
    {'sentence1': Value(dtype='string', id=None),
    'sentence2': Value(dtype='string', id=None),
    'label': ClassLabel(num_classes=2, names=['negative', 'positive'], names_file=None, id=None),
    'idx': Value(dtype='int64', id=None)}


Flattening columns: ``flatten``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

A column type can be a nested struct of several types.
For example a column "answers" may have two subfields "answer_start" and "text".
In this case if you want each of the two subfields to be actual columns, you can use :func:`datasets.Dataset.flatten`:

.. code-block::

    >>> squad = load_dataset("squad", split="train")
    >>> squad
    Dataset({
        features: ['id', 'title', 'context', 'question', 'answers'],
        num_rows: 87599
    })
    >>> flattened_squad = squad.flatten()
    >>> flattened_squad
    Dataset({
        features: ['answers.answer_start', 'answers.text', 'context', 'id', 'question', 'title'],
        num_rows: 87599
    })



Processing data with ``map``
--------------------------------

All the methods we've seen up to now operate on examples taken as a whole and don't inspect (excepted for the ``filter`` method) or modify the content of the samples.

We now turn to the :func:`datasets.Dataset.map` method which is a powerful method inspired by ``tf.data.Dataset`` map method and which you can use to apply a processing function to each example in a dataset, independently or in batch and even generate new rows or columns.

:func:`datasets.Dataset.map` takes a callable accepting a dict as argument (same dict as returned by :obj:`dataset[i]`) and iterates over the dataset by calling the function with each example.

Let's print the length of the ``sentence1`` value for each sample in our dataset:

.. code-block::

    >>> small_dataset = dataset.select(range(10))
    >>> small_dataset
    Dataset(schema: {'sentence1': 'string', 'sentence2': 'string', 'label': 'int64', 'idx': 'int32'}, num_rows: 10)
    >>> small_dataset.map(lambda example: print(len(example['sentence1'])), verbose=False)
    103
    89
    105
    119
    105
    97
    88
    54
    85
    108
    Dataset(schema: {'sentence1': 'string', 'sentence2': 'string', 'label': 'int64', 'idx': 'int32'}, num_rows: 10)

This is basically the same as doing

.. code-block::

    for example in dataset:
        function(example)

The above example had no effect on the dataset because the method we supplied to :func:`datasets.Dataset.map` didn't return a :obj:`dict` or a :obj:`abc.Mapping` that could be used to update the examples in the dataset.

In such a case, :func:`datasets.Dataset.map` will return the original dataset (:obj:`self`) and the user is usually only interested in side effects of the provided method.

Now let's see how we can use a method that actually modifies the dataset with :func:`datasets.Dataset.map`.

Processing data row by row
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The main interest of :func:`datasets.Dataset.map` is to update and modify the content of the table and leverage smart caching and fast backend.

To use :func:`datasets.Dataset.map` to update elements in the table you need to provide a function with the following signature: :obj:`function(example: dict) -> dict`.

Let's add a prefix ``'My sentence: '`` to each ``sentence1`` value in our small dataset:

.. code-block::

    >>> def add_prefix(example):
    ...     example['sentence1'] = 'My sentence: ' + example['sentence1']
    ...     return example
    ... 
    >>> updated_dataset = small_dataset.map(add_prefix)
    >>> updated_dataset['sentence1'][:5]
    ['My sentence: Amrozi accused his brother , whom he called " the witness " , of deliberately distorting his evidence .',
     "My sentence: Yucaipa owned Dominick 's before selling the chain to Safeway in 1998 for $ 2.5 billion .",
     'My sentence: They had published an advertisement on the Internet on June 10 , offering the cargo for sale , he added .',
     'My sentence: Around 0335 GMT , Tab shares were up 19 cents , or 4.4 % , at A $ 4.56 , having earlier set a record high of A $ 4.57 .',
    ]

This call to :func:`datasets.Dataset.map` computed and returned an updated table.

.. note::

    Calling :func:`datasets.Dataset.map` also stored the updated table in a cache file indexed by the current state and the mapped function.
    A subsequent call to :func:`datasets.Dataset.map` (even in another python session) will reuse the cached file instead of recomputing the operation.
    You can test this by running again the previous cell, you will see that the result is directly loaded from the cache and not re-computed again.

The function you provide to :func:`datasets.Dataset.map` should accept an input with the format of an item of the dataset: :obj:`function(dataset[0])` and return a python dict.

The columns and type of the outputs **can be different** from columns and type of the input dict. In this case the new keys will be **added** as additional columns in the dataset.

Each dataset example dict is updated with the dictionary returned by the function. Under the hood :obj:`map` operates like this:

.. code-block::

    new_dataset = []
    for example in dataset:
        processed_example = function(example)
        example.update(processed_example)
        new_dataset.append(example)
    return new_dataset

Since the input example dict is **updated** with output dict generated by our :obj:`add_prefix` function, we could have actually just returned the updated ``sentence1`` field, instead of the full example which is simpler to write:

.. code-block::

    >>> updated_dataset = small_dataset.map(lambda example: {'sentence1': 'My sentence: ' + example['sentence1']})
    >>> updated_dataset['sentence1'][:5]
    ['My sentence: Amrozi accused his brother , whom he called " the witness " , of deliberately distorting his evidence .',
     "My sentence: Yucaipa owned Dominick 's before selling the chain to Safeway in 1998 for $ 2.5 billion .", 'My sentence: They had published an advertisement on the Internet on June 10 , offering the cargo for sale , he added .',
     'My sentence: Around 0335 GMT , Tab shares were up 19 cents , or 4.4 % , at A $ 4.56 , having earlier set a record high of A $ 4.57 .',
     'My sentence: The stock rose $ 2.11 , or about 11 percent , to close Friday at $ 21.51 on the New York Stock Exchange .']

If a dataset was formatted using :func:`datasets.Dataset.set_format`, then:

- if a format type was set, then the format type doesn't change
- if a list of columns that :func:`datasets.Dataset.__getitem__` should return was set, then the new columns added by map are added to this list

Removing columns
^^^^^^^^^^^^^^^^^^^^^^^^

This process of **updating** the original example with the output of the mapped function is simpler to write when mostly adding new columns to a dataset but we need an additional mechanism to easily remove columns.


To this aim, the :obj:`remove_columns=List[str]` argument can be used and provided with a single name or a list of names of columns which should be removed during the :func:`datasets.Dataset.map` operation.

Columns to remove are removed **after** the example has been provided to the mapped function so that the mapped function can use the content of these columns before they are removed.

Here is an example removing the ``sentence1`` column while adding a ``new_sentence`` column with the content of the ``new_sentence``. Said more simply, we are renaming the ``sentence1`` column as ``new_sentence``:

.. code-block::

    >>> updated_dataset = small_dataset.map(lambda example: {'new_sentence': example['sentence1']}, remove_columns=['sentence1'])
    >>> updated_dataset.column_names
    ['sentence2', 'label', 'idx', 'new_sentence']


Using row indices
^^^^^^^^^^^^^^^^^^^^^^

When the argument :obj:`with_indices` is set to :obj:`True`, the indices of the rows (from ``0`` to ``len(dataset)``) will be provided to the mapped function. This function must then have the following signature: :obj:`function(example: dict, indice: int) -> Union[None, dict]`.

In the following example, we add the index of the example as a prefix to the ``sentence2`` field of each example:

.. code-block::

    >>> updated_dataset = small_dataset.map(lambda example, idx: {'sentence2': f'{idx}: ' + example['sentence2']}, with_indices=True)
    >>> updated_dataset['sentence2'][:5]
    ['0: Referring to him as only " the witness " , Amrozi accused his brother of deliberately distorting his evidence .',
     "1: Yucaipa bought Dominick 's in 1995 for $ 693 million and sold it to Safeway for $ 1.8 billion in 1998 .",
     "2: On June 10 , the ship 's owners had published an advertisement on the Internet , offering the explosives for sale .",
     '3: Tab shares jumped 20 cents , or 4.6 % , to set a record closing high at A $ 4.57 .', 
     '4: PG & E Corp. shares jumped $ 1.63 or 8 percent to $ 21.03 on the New York Stock Exchange on Friday .']


Processing data in batches
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

:func:`datasets.Dataset.map` can also work with batches of examples (slices of the dataset).

This is particularly interesting if you have a mapped function which can efficiently handle batches of inputs like the tokenizers of the fast `HuggingFace tokenizers library <https://github.com/huggingface/tokenizers>`__.

To operate on batch of example, just set :obj:`batched=True` when calling :func:`datasets.Dataset.map` and provide a function with the following signature: :obj:`function(examples: Dict[List]) -> Dict[List]` or, if you use indices (:obj:`with_indices=True`): :obj:`function(examples: Dict[List], indices: List[int]) -> Dict[List])`.

In other words, the mapped function should accept an input with the format of a slice of the dataset: :obj:`function(dataset[:10])`.

Let's take an example with a fast tokenizer of the 🤗Transformers library.

First install this library if you haven't already done it:

.. code-block::

    pip install transformers

Then we will import a fast tokenizer, for instance the tokenizer of the Bert model:

.. code-block::

    >>> from transformers import BertTokenizerFast
    >>> tokenizer = BertTokenizerFast.from_pretrained('bert-base-cased')

Now let's batch tokenize the ``sentence1`` fields of our dataset. The tokenizers of the 🤗Transformers library can accept lists of texts as inputs and tokenize them efficiently in batch (for the fast tokenizers in particular).

For more details on the tokenizers of the 🤗Transformers library please refer to its `guide on processing data <https://huggingface.co/transformers/preprocessing.html>`__.

This tokenizer will output a dictionary-like object with three fields: ``input_ids``, ``token_type_ids``, ``attention_mask`` corresponding to Bert model's required inputs. Each field contains a list (batch) of samples.

The output of the tokenizer is thus compatible with the :func:`datasets.Dataset.map` method which is also expected to return a dictionary. We can thus directly return the dictionary generated by the tokenizer as the output of our mapped function:

.. code-block::

    >>> encoded_dataset = dataset.map(lambda examples: tokenizer(examples['sentence1']), batched=True)
    >>> encoded_dataset.column_names
    ['sentence1', 'sentence2', 'label', 'idx', 'input_ids', 'token_type_ids', 'attention_mask']
    >>> encoded_dataset[0]
    {'sentence1': 'Amrozi accused his brother , whom he called " the witness " , of deliberately distorting his evidence .',
     'sentence2': 'Referring to him as only " the witness " , Amrozi accused his brother of deliberately distorting his evidence .',
     'label': 1,
     'idx': 0,
     'input_ids': [  101,  7277,  2180,  5303,  4806,  1117,  1711,   117,  2292, 1119,  1270,   107,  1103,  7737,   107,   117,  1104,  9938, 4267, 12223, 21811,  1117,  2554,   119,   102],
     'token_type_ids': [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0],
     'attention_mask': [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1]
    }

We have indeed added the columns for ``input_ids``, ``token_type_ids``, ``attention_mask`` which contain the encoded version of the ``sentence1`` field.

The batch size provided to the mapped function can be controlled by the :obj:`batch_size` argument. The default value is ``1000``, i.e. batches of 1000 samples will be provided to the mapped function by default.

Multiprocessing
---------------------------

Multiprocessing can speed up significantly the processing of your dataset.

The :func:`datasets.Dataset.map` method has an argument ``num_proc`` that allows you to set the number of processes to use.

In this case, each process takes care of processing one shard of the dataset and all the processes are ran in parallel.

Augmenting the dataset
---------------------------

Using :func:`datasets.Dataset.map` in batched mode (i.e. with :obj:`batched=True`) actually let you control the size of the generated dataset freely.

More precisely, in batched mode :func:`datasets.Dataset.map` will provide a batch of examples (as a dict of lists) to the mapped function and expect the mapped function to return back a batch of examples (as a dict of lists) but **the input and output batch are not required to be of the same size**.

In other words, a batch mapped function can take as input a batch of size ``N`` and return a batch of size ``M`` where ``M`` can be greater or less than ``N`` and can even be zero.

The resulting dataset can thus have a different size from the original dataset.

This can be taken advantage of for several use-cases:

- the :func:`datasets.Dataset.filter` method makes use of variable size batched mapping under the hood to change the size of the dataset and filter some columns,
- it's possible to cut examples which are too long in several snippets,
- it's also possible to do data augmentation on each example.

.. note::

    **One important condition on the output of the mapped function.** Each field in the output dictionary returned by the mapped function must contain the **same number of elements** as the other field in this output dictionary otherwise it's not possible to define the number of examples in the output returned the mapped function. This number can vary between the successive batches processed by the mapped function but in a single batch, all fields of the output dictionary should have the same number of elements.

Let's show how we can implemented the two simple examples we mentioned: "cutting examples which are too long in several snippets" and do some "data augmentation".

We'll start by chunking the ``sentence1`` field of our dataset in chunks of 50 characters and stack all these chunks to make our new dataset.

We will also remove all the columns of the dataset and only keep the chunks in order to avoid the issue of uneven field lengths mentioned in the above note (we could also duplicate the other fields to compensated but let's make it as simple as possible here):

.. code-block::

    >>> def chunk_examples(examples):
    ...     chunks = []
    ...     for sentence in examples['sentence1']:
    ...         chunks += [sentence[i:i + 50] for i in range(0, len(sentence), 50)]
    ...     return {'chunks': chunks}
    ... 
    >>> chunked_dataset = dataset.map(chunk_examples, batched=True, remove_columns=dataset.column_names)
    >>> chunked_dataset
    Dataset(schema: {'chunks': 'string'}, num_rows: 10470)
    >>> chunked_dataset[:10]
    {'chunks': ['Amrozi accused his brother , whom he called " the ',
                'witness " , of deliberately distorting his evidenc',
                'e .',
                "Yucaipa owned Dominick 's before selling the chain",
                ' to Safeway in 1998 for $ 2.5 billion .',
                'They had published an advertisement on the Interne',
                't on June 10 , offering the cargo for sale , he ad',
                'ded .',
                'Around 0335 GMT , Tab shares were up 19 cents , or',
                ' 4.4 % , at A $ 4.56 , having earlier set a record']}

As we can see, our dataset is now much longer (10470 row) and contains a single column with chunks of 50 characters. Some chunks are smaller since they are the last part of the sentences which were smaller than 50 characters. We could then filter them with :func:`datasets.Dataset.filter` for instance.

Now let's finish with the other example and try to do some data augmentation. We will use a Roberta model to sample some masked tokens.

Here we can use the `FillMaskPipeline of 🤗Transformers <https://huggingface.co/transformers/main_classes/pipelines.html?#transformers.FillMaskPipeline>`__ to generate options for a masked token in a sentence.

We will randomly select a word to mask in the sentence and return the original sentence plus the two top replacements by Roberta.

Since the Roberta model is quite large to run on a small laptop CPU, we will restrict this example to a small dataset of 100 examples and we will lower the batch size to be able to follow the processing more precisely.

.. code-block::

    >>> from random import randint
    >>> from transformers import pipeline
    >>> 
    >>> fillmask = pipeline('fill-mask')
    >>> mask_token = fillmask.tokenizer.mask_token
    >>> smaller_dataset = dataset.filter(lambda e, i: i<100, with_indices=True)
    >>> 
    >>> def augment_data(examples):
    ...     outputs = []
    ...     for sentence in examples['sentence1']:
    ...         words = sentence.split(' ')
    ...         K = randint(1, len(words)-1)
    ...         masked_sentence = " ".join(words[:K]  + [mask_token] + words[K+1:])
    ...         predictions = fillmask(masked_sentence)
    ...         augmented_sequences = [predictions[i]['sequence'] for i in range(3)]
    ...         outputs += [sentence] + augmented_sequences
    ...     
    ...     return {'data': outputs}
    ... 
    >>> augmented_dataset = smaller_dataset.map(augment_data, batched=True, remove_columns=dataset.column_names, batch_size=8)
    >>> len(augmented_dataset)
    400
    >>> augmented_dataset[:9]['data']
    ['Amrozi accused his brother , whom he called " the witness " , of deliberately distorting his evidence .',
     'Amrozi accused his brother, whom he called " the witness ", of deliberately withholding his evidence.',
     'Amrozi accused his brother, whom he called " the witness ", of deliberately suppressing his evidence.',
     'Amrozi accused his brother, whom he called " the witness ", of deliberately destroying his evidence.',
     "Yucaipa owned Dominick 's before selling the chain to Safeway in 1998 for $ 2.5 billion .",
     'Yucaipa owned Dominick Stores before selling the chain to Safeway in 1998 for $ 2.5 billion.',
     "Yucaipa owned Dominick's before selling the chain to Safeway in 1998 for $ 2.5 billion.", 
     'Yucaipa owned Dominick Pizza before selling the chain to Safeway in 1998 for $ 2.5 billion.']

Here we have now multiplied the size of our dataset by ``4`` by adding three alternatives generated with Roberta to each example. We can see that the word ``distorting`` in the first example was augmented with other possibilities by the Roberta model: ``withholding``, ``suppressing``, ``destroying``, while in the second sentence, it was the ``'s`` token which was randomly sampled and replaced by ``Stores`` and ``Pizza``.

Obviously this is a very simple example for data augmentation and it could be improved in several ways, the most interesting take-away is probably how this can be written in roughtly ten lines of code without any loss in flexibility.

Processing several splits at once
-----------------------------------

When you load a dataset that has various splits, :func:`datasets.load_dataset` returns a :obj:`datasets.DatasetDict` that is a dictionary with split names as keys ('train', 'test' for example), and :obj:`datasets.Dataset` objects as values.
You can directly call map, filter, shuffle, and sort directly on a :obj:`datasets.DatasetDict` object:

.. code-block::

    >>> from datasets import load_dataset
    >>>
    >>> dataset = load_dataset('glue', 'mrpc')  # load all the splits
    >>> dataset.keys()
    dict_keys(['train', 'validation', 'test'])
    >>> encoded_dataset = dataset.map(lambda examples: tokenizer(examples['sentence1']), batched=True) 
    >>> encoded_dataset["train"][0]
    {'sentence1': 'Amrozi accused his brother , whom he called " the witness " , of deliberately distorting his evidence .',
     'sentence2': 'Referring to him as only " the witness " , Amrozi accused his brother of deliberately distorting his evidence .',
     'label': 1,
     'idx': 0,
     'input_ids': [  101,  7277,  2180,  5303,  4806,  1117,  1711,   117,  2292, 1119,  1270,   107,  1103,  7737,   107,   117,  1104,  9938, 4267, 12223, 21811,  1117,  2554,   119,   102],
     'token_type_ids': [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0],
     'attention_mask': [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1]
    }

This concludes our chapter on data processing with 🤗Datasets (and 🤗Transformers).

Concatenate several datasets
----------------------------

When you have several :obj:`datasets.Dataset` objects that share the same column types, you can create a new :obj:`datasets.Dataset` object that is the concatenation of them:

.. code-block::

    >>> from datasets import concatenate_datasets, load_dataset
    >>>
    >>> bookcorpus = load_dataset("bookcorpus", split="train")
    >>> wiki = load_dataset("wikipedia", "20200501.en", split="train")
    >>> wiki = wiki.remove_columns("title")  # only keep the text
    >>>
    >>> assert bookcorpus.features.type == wiki.features.type
    >>> bert_dataset = concatenate_datasets([bookcorpus, wiki])

If you want to interleave the datasets instead of concatenating them, you can use :func:`datasets.interleave_datasets`.


Saving a processed dataset on disk and reload it
------------------------------------------------

Once you have your final dataset you can save it on your disk and reuse it later using :obj:`datasets.load_from_disk`.
Saving a dataset creates a directory with various files:

- arrow files: they contain your dataset's data
- dataset_info.json: contains the description, citations, etc. of the dataset
- state.json: contains the list of the arrow files and other informations like the dataset format type, if any (torch or tensorflow for example)

.. code-block::

    >>> encoded_dataset.save_to_disk("path/of/my/dataset/directory")
    >>> ...
    >>> from datasets import load_from_disk
    >>> reloaded_encoded_dataset = load_from_disk("path/of/my/dataset/directory")

Both :obj:`datasets.Dataset` and :obj:`datasets.DatasetDict` objects can be saved on disk, by using respectively :func:`datasets.Dataset.save_to_disk` and :func:`datasets.DatasetDict.save_to_disk`.

Furthermore it is also possible to save :obj:`datasets.Dataset` and :obj:`datasets.DatasetDict` to other filesystems and cloud storages such as S3 by using respectively :func:`datasets.Dataset.save_to_disk` 
and :func:`datasets.DatasetDict.save_to_disk` and providing a ``Filesystem`` as input ``fs``. To learn more about saving your ``datasets`` to other filesystem take a look at :doc:`filesystems`.

Exporting a dataset to csv, or to python objects
------------------------------------------------

You can save your dataset in csv format using :func:`datasets.Dataset.to_csv`, so that you can use your dataset in other applications if you want to.

To get directly python objects, you can use :func:`datasets.Dataset.to_pandas` or :func:`datasets.Dataset.to_dict` to export the dataset as a pandas DataFrame or a python dict.

Controling the cache behavior
-----------------------------------

When applying transforms on a dataset, the data are stored in cache files.
The caching mechanism allows to reload an existing cache file if it's already been computed.

Reloading a dataset is possible since the cache files are named using the dataset fingerprint, which is updated after each transform.

Note that the caching extends beyond sessions. Re-running the very same dataset processing methods (in the same order and on the same data files) in a different session will load from the same cache files.
This is possible thanks to a custom hashing function that works with most python objects (see fingerprinting section below).


Fingerprinting
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The fingerprint of a dataset in a given state is an internal value computed by combining the fingerprint of the previous state and a hash of the latest transform that was applied. (Transforms are all the processing method for transforming a dataset that we listed in this chapter (:func:`datasets.Dataset.map`, :func:`datasets.Dataset.shuffle`, etc)
The initial fingerprint is computed using a hash of the arrow table, or a hash of the arrow files if the dataset lives on disk.

For example:

.. code-block::

    >>> from datasets import Dataset
    >>> dataset1 = Dataset.from_dict({"a": [0, 1, 2]})
    >>> dataset2 = dataset1.map(lambda x: {"a": x["a"] + 1})
    >>> print(dataset1._fingerprint, dataset2._fingerprint)
    d19493523d95e2dc 5b86abacd4b42434

The new fingerprint is a combination of the previous fingerprint and the hash of the given transform. For a transform to be hashable, it needs to be picklable using dill or pickle. In particular for :func:`datasets.Dataset.map`, you need to provide a picklable processing method to apply on the dataset so that a determinist fingerprint can be computed by hashing the full state of the provided method (the fingerprint is computed taking into account all the dependencies of the method you provide). 
For non-hashable transform, a random fingerprint is used and a warning is raised.
Make sure your transforms and parameters are serializable with pickle or dill for the dataset fingerprinting and caching to work.
If you reuse a non-hashable transform, the caching mechanism will consider it to be different from the previous calls and recompute everything.

Enable or disable caching
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Locally you can prevent the library from reloading a cached file by using ``load_from_cache=False`` in transforms like :func:`datasets.Dataset.map` for example.
You can also specify the name of path where the cache file will be written using the parameter ``cache_file_name``.

It is also possible to disable caching globally with :func:`datasets.set_caching_enabled`.

If the caching is disabled, the library will no longer reload cached dataset files when applying transforms to the datasets.
More precisely, if the caching is disabled:
- cache files are always recreated
- cache files are written to a temporary directory that is deleted when session closes
- cache files are named using a random hash instead of the dataset fingerprint
- use :func:`datasets.Dataset.save_to_disk` to save a transformed dataset or it will be deleted when session closes
- caching doesn't affect :func:`datasets.load_dataset`. If you want to regenerate a dataset from scratch you should use
the ``download_mode`` parameter in :func:`datasets.load_dataset`.

To disable caching you can run:

.. code-block::

    >>> from datasets import set_caching_enabled
    >>> set_caching_enabled(False)

You can also query the current status of the caching with :func:`datasets.is_caching_enabled`:

Mapping in a distributed setting
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

In a distributed setting, you may use caching and a :func:`torch.distributed.barrier` to make sure that only the main process performs the mapping, while the other ones load its results. This avoids duplicating work between all the processes, or worse, requesting more CPUs than your system can handle. For example:

.. code-block::

    >>> from datasets import Dataset
    >>> import torch.distributed
    >>> 
    >>> dataset1 = Dataset.from_dict({"a": [0, 1, 2]})
    >>> 
    >>> if training_args.local_rank > 0:
    ...     print("Waiting for main process to perform the mapping")
    ...     torch.distributed.barrier()
    >>> 
    >>> dataset2 = dataset1.map(lambda x: {"a": x["a"] + 1})
    >>> 
    >>> if training_args.local_rank == 0:
    ...     print("Loading results from main process")
    ...     torch.distributed.barrier()

When it encounters a barrier, each process will stop until all other processes have reached the barrier. The non-main processes reach the barrier first, before the mapping, and wait there. The main processes creates the cache for the processed dataset. It then reaches the barrier, at which point the other processes resume, and load the cache instead of performing the processing themselves.
