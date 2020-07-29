:desc: Information about changes between major versions of chatbot framework
       Rasa Core and how you can migrate from one version to another.

.. _migration-guide:

Migration Guide
===============

.. edit-link::

This page contains information about changes between major versions and
how you can migrate from one version to another.

.. _migration-to-rasa-2.0:

Rasa 1.10 to Rasa 2.0
---------------------

General
~~~~~~~
- The deprecated brokers ``FileProducer``, ``KafkaProducer``, ``PikaProducer``
  and the ``SQLProducer`` have been removed. If you used these brokers in your
  ``endpoints.yml`` make sure to use the renamed variants instead:
  - ``FileProducer`` became ``FileEventBroker``
  - ``KafkaProducer`` became ``KafkaEventBroker``
  - ``PikaProducer`` became ``PikaEventBroker``
  - ``SQLProducer`` became  ``SQLEventBroker``

- The deprecated ``EmbeddingIntentClassifier`` has been removed. If you used this
  component in your ``pipeline`` configuration (``config.yml``) you can replace it
  with ``DIETClassifier``. It accepts the same configuration parameters.

- The deprecated ``KerasPolicy`` has been removed. If you used this
  component in your ``policies`` configuration (``config.yml``) you can replace it
  with ``TEDPolicy``. It accepts the same configuration parameters.

RulePolicy
~~~~~~~~~~

With the introduction of :ref:`rules`, the following policies are deprecated:

- :ref:`mapping-policy`
- :ref:`fallback-policy`
- :ref:`two-stage-fallback-policy`
- :ref:`form-policy`

.. _migrate-mapping-policy-to-rule-policy:

Migrating from the Mapping Policy
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

If you previously used the :ref:`mapping-policy`, you can follow the documentation on
:ref:`rules-faqs` to convert your mapped intents to rules. Suppose you previously
mapped an intent ``ask_is_bot`` as follows:

.. code-block:: yaml

    intents:
     - ask_is_bot:
         triggers: action_is_bot

This becomes the following rule:

.. code-block:: yaml

    rules:
        rule: Rule to map `ask_is_bot` intent
        steps:
        - intent: ask_is_bot
        - action: action_is_bot

.. _migrate-fallback-policy-to-rule-policy:

Migrating from the Fallback Policy
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

If you previously used the :ref:`fallback-policy`, the following model
configuration would translate as follows given a previous configuration like this:

.. code-block:: yaml

    policies:
      - name: "FallbackPolicy"
        nlu_threshold: 0.4
        core_threshold: 0.3
        fallback_action_name: "action_default_fallback"
        ambiguity_threshold: 0.1

The new configuration would then look like:

.. code-block:: yaml

    policies:
    # Other policies
    - name: RulePolicy
      core_fallback_threshold: 0.3
      fallback_action_name: "action_default_fallback"

    pipeline:
      # Other components
      - name: FallbackClassifier
        threshold: 0.4
        ambiguity_threshold: 0.1

In addition, you need to add a :ref:`rule<rules>` to specify which action to run in case
of low NLU confidence:

.. code-block:: yaml

    rules:

    - rule: Ask the user to rephrase whenever they send a message with low NLU confidence
      steps:
      - intent: nlu_fallback
      - action: utter_please_rephrase

Please see the :ref:`fallback-actions` documentation for more information.

.. _migrate-two-stage-fallback-policy-to-rule-policy:

Migrating from the Two-Stage-Fallback Policy
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

If you previously used the :ref:`two-stage-fallback-policy`, the following model
configuration would translate as follows given a previous configuration like this:

    .. code-block:: yaml

        policies:
          - name: TwoStageFallbackPolicy
            nlu_threshold: 0.4
            ambiguity_threshold: 0.1
            core_threshold: 0.3
            fallback_core_action_name: "action_default_fallback"
            fallback_nlu_action_name: "action_default_fallback"
            deny_suggestion_intent_name: "out_of_scope"

The new configuration would then look like:

.. code-block:: yaml

    policies:
    # Other policies
    - name: RulePolicy
      core_fallback_threshold: 0.3
      fallback_action_name: "action_default_fallback"

    pipeline:
      # Other components
      - name: FallbackClassifier
        threshold: 0.4
        ambiguity_threshold: 0.1

In addition, you need to add a :ref:`rule<rules>` to activate the Two-Stage Fallback for
messages with low NLU confidence.

.. code-block:: yaml

    - rule: Implementation of the TwoStageFallbackPolicy
      steps:
      # This intent is automatically triggered by the `FallbackClassifier` in the NLU
      # pipeline in case the intent confidence was below the specified threshold.
      - intent: nlu_fallback
      # The Fallback is now implemented as a form.
      - action: two_stage_fallback
      - form: two_stage_fallback

Note that the previous parameters ``fallback_nlu_action_name`` and
``deny_suggestion_intent_name`` are no longer configurable and have the fixed values
``action_default_fallback`` and ``out_of_scope``.

Please see the :ref:`fallback-actions` documentation for more information.

FormPolicy
~~~~~~~~~~

As of Rasa Open Source 2.0, the logic for :ref:`forms` has been moved from the Rasa SDK
to Rasa Open Source to simplify implementations of custom action libraries. This means
you no longer need to use the ``FormAction`` when implementing custom actions with the
Python SDK. Instead you can use a regular ``Action`` to validate and request slots.

It is now recommended to move the slot mappings from your custom action to the
:ref:`domain<domains>` of your bot. Consider a custom form action like this:

.. code-block:: python

    class RestaurantForm(FormAction):
        def name(self) -> Text:
            return "restaurant_form"

        @staticmethod
        def required_slots(tracker: Tracker) -> List[Text]:
            return ["cuisine"]

        def slot_mappings(self) -> Dict[Text, Union[Dict, List[Dict]]]:
            return {
                "cuisine": self.from_entity(entity="cuisine", not_intent="chitchat"),
            }

        @staticmethod
        def cuisine_db() -> List[Text]:
            """Database of supported cuisines"""

            return ["caribbean", "chinese", "french"]

        def validate_cuisine(
            self,
            value: Text,
            dispatcher: CollectingDispatcher,
            tracker: Tracker,
            domain: Dict[Text, Any],
        ) -> Dict[Text, Any]:
            """Validate cuisine value."""

            if value.lower() in self.cuisine_db():
                # validation succeeded, set the value of the "cuisine" slot to value
                return {"cuisine": value}
            else:
                dispatcher.utter_message(template="utter_wrong_cuisine")
                # validation failed, set this slot to None, meaning the
                # user will be asked for the slot again
                return {"cuisine": None}

        def submit(
            self,
            dispatcher: CollectingDispatcher,
            tracker: Tracker,
            domain: Dict[Text, Any],
        ) -> List[Dict]:
            """Define what the form has to do
                after all required slots are filled"""

            # utter submit template
            dispatcher.utter_message(template="utter_submit")
            return []

Start the migration by adding the :ref:`rule-policy` to your model configuration:

.. code-block:: yaml

    policies:
    # Other policies
    # ...
    - name: RulePolicy

Then you need to define the form and the required slots in the domain as described in
:ref:`forms-domain`:

.. code-block:: yaml

  forms:
    restaurant_form:
      cuisine:
      - type: cuisine
        entity: cuisine
        not_intent: chitchat

You don't have to add a rule for activating the form as this is already covered by your
existing stories. However, you have to add a story for handle the submission of the
form.

.. code-block:: yaml

    - rule: Submit form
      condition:
      # Condition that form is active.
      - form: restaurant_form
      steps:
      - action: restaurant_form
      - form: null
      - slot: requested_slot
        value: null
      # The action we want to run when the form is submitted.
      - action: utter_submit

The last step is to implement a custom action to validate the form slots. Start by
adding the custom action to your domain:

.. code-block:: yaml

    actions:
    # Other actions
    # ...
    - validate_restaurant_form

Then add a custom action which validates the ``cuisine`` slot:

.. code-block:: python

    class RestaurantFormValidator(Action):
        def name(self) -> Text:
            return "validate_restaurant_form"

        def run(
            self, dispatcher: CollectingDispatcher, tracker: Tracker, domain: Dict
        ) -> List[EventType]:
            extracted_slots: Dict[Text, Any] = tracker.get_extracted_slots()

            cuisine_slot_value = extracted_slots.get("cuisine")
            validated_slot_event = self.validate_cuisine(
                cuisine_slot_value, dispatcher, tracker, domain
            )
            return [validated_slot_event]

        @staticmethod
        def cuisine_db() -> List[Text]:
            """Database of supported cuisines"""

            return ["caribbean", "chinese", "french"]

        def validate_cuisine(
            self,
            value: Text,
            dispatcher: CollectingDispatcher,
            tracker: Tracker,
            domain: Dict[Text, Any],
        ) -> EventType:
            """Validate cuisine value."""

            if value.lower() in self.cuisine_db():
                # validation succeeded, set the value of the "cuisine" slot to value
                return SlotSet("cuisine", value)
            else:
                dispatcher.utter_message(template="utter_wrong_cuisine")
                # validation failed, set this slot to None, meaning the
                # user will be asked for the slot again
                return SlotSet("cuisine", None)

Please see :ref:`forms` if you have further customizations in your ``FormAction``.

.. _migration-to-rasa-1.8:

Rasa 1.7 to Rasa 1.8
--------------------
.. warning::

  This is a release **breaking backwards compatibility**.
  It is not possible to load previously trained models. Please make sure to retrain a
  model before trying to use it with this improved version.

General
~~~~~~~
- The :ref:`ted_policy` replaced the ``keras_policy`` as recommended machine
  learning policy. New projects generated with ``rasa init`` will automatically use
  this policy. In case you want to change your existing model configuration to use the
  :ref:`ted_policy` add this to the ``policies`` section in your ``config.yml``
  and remove potentially existing ``KerasPolicy`` entries:

  .. code-block:: yaml

    policies:
    # - ... other policies
    - name: TEDPolicy
      max_history: 5
      epochs: 100

  The given snippet specifies default values for the parameters ``max_history`` and
  ``epochs``. ``max_history`` is particularly important and strongly depends on your stories.
  Please see the docs of the :ref:`ted_policy` if you want to customize them.

- All pre-defined pipeline templates are deprecated. **Any templates you use will be
  mapped to the new configuration, but the underlying architecture is the same**.
  Take a look at :ref:`choosing-a-pipeline` to decide on what components you should use
  in your configuration file.

- The :ref:`embedding_policy` was renamed to :ref:`ted_policy`. The functionality of the policy stayed the same.
  Please update your configuration files to use ``TEDPolicy`` instead of ``EmbeddingPolicy``.

- Most of the model options for ``EmbeddingPolicy``, ``EmbeddingIntentClassifier``, and ``ResponseSelector`` got
  renamed. Please update your configuration files using the following mapping:

  =============================  =======================================================
  Old model option               New model option
  =============================  =======================================================
  hidden_layers_sizes_a          dictionary "hidden_layers_sizes" with key "text"
  hidden_layers_sizes_b          dictionary "hidden_layers_sizes" with key "label"
  hidden_layers_sizes_pre_dial   dictionary "hidden_layers_sizes" with key "dialogue"
  hidden_layers_sizes_bot        dictionary "hidden_layers_sizes" with key "label"
  num_transformer_layers         number_of_transformer_layers
  num_heads                      number_of_attention_heads
  max_seq_length                 maximum_sequence_length
  dense_dim                      dense_dimension
  embed_dim                      embedding_dimension
  num_neg                        number_of_negative_examples
  mu_pos                         maximum_positive_similarity
  mu_neg                         maximum_negative_similarity
  use_max_sim_neg                use_maximum_negative_similarity
  C2                             regularization_constant
  C_emb                          negative_margin_scale
  droprate_a                     droprate_dialogue
  droprate_b                     droprate_label
  evaluate_every_num_epochs      evaluate_every_number_of_epochs
  evaluate_on_num_examples       evaluate_on_number_of_examples
  =============================  =======================================================

  Old configuration options will be mapped to the new names, and a warning will be thrown.
  However, these will be deprecated in a future release.

- The Embedding Intent Classifier is now deprecated and will be replaced by :ref:`DIETClassifier <diet-classifier>`
  in the future.
  ``DIETClassfier`` performs intent classification as well as entity recognition.
  If you want to get the same model behavior as the current ``EmbeddingIntentClassifier``, you can use
  the following configuration of ``DIETClassifier``:

  .. code-block:: yaml

    pipeline:
    # - ... other components
    - name: DIETClassifier
      hidden_layers_sizes:
        text: [256, 128]
      number_of_transformer_layers: 0
      weight_sparsity: 0
      intent_classification: True
      entity_recognition: False
      use_masked_language_model: False
      BILOU_flag: False
      # ... any other parameters

  See :ref:`DIETClassifier <diet-classifier>` for more information about the new component.
  Specifying ``EmbeddingIntentClassifier`` in the configuration maps to the above component definition, the
  behavior is unchanged from previous versions.

- ``CRFEntityExtractor`` is now deprecated and will be replaced by ``DIETClassifier`` in the future. If you want to
  get the same model behavior as the current ``CRFEntityExtractor``, you can use the following configuration:

  .. code-block:: yaml

    pipeline:
    # - ... other components
    - name: LexicalSyntacticFeaturizer
      features: [
        ["low", "title", "upper"],
        [
          "BOS",
          "EOS",
          "low",
          "prefix5",
          "prefix2",
          "suffix5",
          "suffix3",
          "suffix2",
          "upper",
          "title",
          "digit",
        ],
        ["low", "title", "upper"],
      ]
    - name: DIETClassifier
      intent_classification: False
      entity_recognition: True
      use_masked_language_model: False
      number_of_transformer_layers: 0
      # ... any other parameters

  ``CRFEntityExtractor`` featurizes user messages on its own, it does not depend on any featurizer.
  We extracted the featurization from the component into the new featurizer :ref:`LexicalSyntacticFeaturizer`. Thus,
  in order to obtain the same results as before, you need to add this featurizer to your pipeline before the
  :ref:`diet-classifier`.
  Specifying ``CRFEntityExtractor`` in the configuration maps to the above component definition, the behavior
  is unchanged from previous versions.

- If your pipeline contains ``CRFEntityExtractor`` and ``EmbeddingIntentClassifier`` you can substitute both
  components with :ref:`DIETClassifier <diet-classifier>`. You can use the following pipeline for that:

  .. code-block:: yaml

    pipeline:
    # - ... other components
    - name: LexicalSyntacticFeaturizer
      features: [
        ["low", "title", "upper"],
        [
          "BOS",
          "EOS",
          "low",
          "prefix5",
          "prefix2",
          "suffix5",
          "suffix3",
          "suffix2",
          "upper",
          "title",
          "digit",
        ],
        ["low", "title", "upper"],
      ]
    - name: DIETClassifier
      number_of_transformer_layers: 0
      # ... any other parameters

.. _migration-to-rasa-1.7:

Rasa 1.6 to Rasa 1.7
--------------------

General
~~~~~~~
- By default, the ``EmbeddingIntentClassifier``, ``EmbeddingPolicy``, and ``ResponseSelector`` will
  now normalize the top 10 confidence results if the ``loss_type`` is ``"softmax"`` (which has been
  default since 1.3, see :ref:`migration-to-rasa-1.3`). This is configurable via the ``ranking_length``
  configuration parameter; to turn off normalization to match the previous behavior, set ``ranking_length: 0``.

.. _migration-to-rasa-1.3:

Rasa 1.2 to Rasa 1.3
--------------------
.. warning::

  This is a release **breaking backwards compatibility**.
  It is not possible to load previously trained models. Please make sure to retrain a
  model before trying to use it with this improved version.

General
~~~~~~~
- Default parameters of ``EmbeddingIntentClassifier`` are changed. See :ref:`components` for details.
  Architecture implementation is changed as well, so **old trained models cannot be loaded**.
  Default parameters and architecture for ``EmbeddingPolicy`` are changed. See :ref:`policies` for details.
  It uses transformer instead of lstm. **Old trained models cannot be loaded**.
  They use ``inner`` similarity and ``softmax`` loss by default instead of
  ``cosine`` similarity and ``margin`` loss (can be set in config file).
  They use ``balanced`` batching strategy by default to counteract class imbalance problem.
  The meaning of ``evaluate_on_num_examples`` is changed. If it is non zero, random examples will be
  picked by stratified split and used as **hold out** validation set, so they will be excluded from training data.
  We suggest to set it to zero (default) if data set contains a lot of unique examples of dialogue turns.
  Removed ``label_tokenization_flag`` and ``label_split_symbol`` from component. Instead moved intent splitting to ``Tokenizer`` components via ``intent_tokenization_flag`` and ``intent_split_symbol`` flag.
- Default ``max_history`` for ``EmbeddingPolicy`` is ``None`` which means it'll use
  the ``FullDialogueTrackerFeaturizer``. We recommend to set ``max_history`` to
  some finite value in order to use ``MaxHistoryTrackerFeaturizer``
  for **faster training**. See :ref:`featurization_conversations` for details.
  We recommend to increase ``batch_size`` for ``MaxHistoryTrackerFeaturizer``
  (e.g. ``"batch_size": [32, 64]``)
- **Compare** mode of ``rasa train core`` allows the whole core config comparison.
  Therefore, we changed the naming of trained models. They are named by config file
  name instead of policy name. Old naming style will not be read correctly when
  creating **compare** plots (``rasa test core``). Please remove old trained models
  in comparison folder and retrain. Normal core training is unaffected.
- We updated the **evaluation metric** for our **NER**. We report the weighted precision and f1-score.
  So far we included ``no-entity`` in this report. However, as most of the tokens actually don't have
  an entity set, this will influence the weighted precision and f1-score quite a bit. From now on we
  exclude ``no-entity`` from the evaluation. The overall metrics now only include proper entities. You
  might see a drop in the performance scores when running the evaluation again.
- ``/`` is reserved as a delimiter token to distinguish between retrieval intent and the corresponding response text
  identifier. Make sure you don't include ``/`` symbol in the name of your intents.

.. _migration-to-rasa-1.0:

Rasa NLU 0.14.x and Rasa Core 0.13.x to Rasa 1.0
------------------------------------------------
.. warning::

  This is a release **breaking backwards compatibility**.
  It is not possible to load previously trained models. Please make sure to retrain a
  model before trying to use it with this improved version.

General
~~~~~~~

- The scripts in ``rasa.core`` and ``rasa.nlu`` can no longer be executed. To train, test, run, ... an NLU or Core
  model, you should now use the command line interface ``rasa``. The functionality is, for the most part, the same as before.
  Some changes in commands reflect the combined training and running of NLU and Core models, but NLU and Core can still
  be trained and used individually. If you attempt to run one of the old scripts in ``rasa.core`` or ``rasa.nlu``,
  an error is thrown that points you to the command you
  should use instead. See all the new commands at :ref:`command-line-interface`.

- If you have written a custom output channel, all ``send_`` methods subclassed
  from the ``OutputChannel`` class need to take an additional ``**kwargs``
  argument. You can use these keyword args from your custom action code or the
  templates in your domain file to send any extra parameters used in your
  channel's send methods.

- If you were previously importing the ``Button`` or ``Element`` classes from
  ``rasa_core.dispatcher``, these are now to be imported from ``rasa_sdk.utils``.

- Rasa NLU and Core previously used `separate configuration files
  <https://legacy-docs.rasa.com/docs/nlu/0.15.1/migrations/?&_ga=2.218966814.608734414.1560704810-314462423.1543594887#id1>`_.
  These two files should be merged into a single file either named ``config.yml``, or passed via the ``--config`` parameter.

Script parameters
~~~~~~~~~~~~~~~~~
- All script parameter names have been unified to follow the same schema.
  Any underscores (``_``) in arguments have been replaced with dashes (``-``).
  For example: ``--max_history`` has been changed to ``--max-history``. You can
  see all of the script parameters in the ``--help`` output of the commands
  in the :ref:`command-line-interface`.

- The ``--num_threads`` parameter was removed from the ``run`` command. The
  server will always run single-threaded, but will now run asynchronously. If you want to
  make use of multiple processes, feel free to check out the `Sanic server
  documentation <https://sanic.readthedocs.io/en/latest/sanic/deploying.html#running-via-gunicorn>`_.

- To avoid conflicts in script parameter names, connectors in the ``run`` command now need to be specified with
  ``--connector``, as ``-c`` is no longer supported. The maximum history in the ``rasa visualize`` command needs to be
  defined with ``--max-history``. Output paths and log files cannot be specified with ``-o`` anymore; ``--out`` and
  ``--log-file`` should be used. NLU data has been standarized to be ``--nlu`` and the name of
  any kind of data files or directory to be ``--data``.

HTTP API
~~~~~~~~
- There are numerous HTTP API endpoint changes which can be found `here <https://rasa.com/docs/rasa/api/http-api/>`_.
