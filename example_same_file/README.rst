Breaking-Question Calibration Datasets
======================================

The purpose of these tests is to calibrate the breaking questions
the goal is to allow an agent to automatically modify the questions as needed

.json files
-----------

there is a .json with expected result for each question
format <name of question enum>.json
each .json should include ``true`` examples
the false is running against all questions

expected results
----------------

100% success over 3-5 iterations
this is a very small sample case

tool to know
------------

- DSPy -- Machine learning / Compilation
- Promptimize / Promptim -- Test-Driven Development (TDD)
- instructor -- structured Output Enforcers

----

Why per-question (not just the verdict)
---------------------------------------

``suite run`` scores only the aggregate ``breaking`` verdict and throws
``info.answers`` away, so it can't tell you *which* question was miscalibrated.
Keeping the per-question answer and comparing it to a gold label separates the
two root causes of a wrong verdict (the Bucket-A / Bucket-B split in
`../../sdk_agents/breaking/DEBUG.md <../../../sdk_agents/breaking/DEBUG.md>`_):

- **Bucket A** -- the agent answered a question *wrong* -> reword it / feed the
  sub-agent better evidence.
- **Bucket B** -- every answer is right but the *verdict* is wrong -> re-tier the
  question or re-sweep the threshold.

File format
-----------

One file per question, named after its ``BreakingQuestion`` member
(`models/breaking_questions.py <../../../models/breaking_questions.py>`_):

.. code-block:: json

   // API_SIGNATURE_CHANGE.json
   [
     // shorthand — a bare bool/null labels the file's own question
     {"path": "<repo-relative patch>", "expected": true},   // should fire
     {"path": "<repo-relative patch>", "expected": false},  // hard negative
     {"path": "<repo-relative patch>", "expected": null},   // honestly can't-determine

     // map — label any answers you know for this patch; one run scores them all.
     // The file's own question (API_SIGNATURE_CHANGE) MUST be present.
     {"path": "<repo-relative patch>",
      "expected": {"API_SIGNATURE_CHANGE": true,
                   "REMOVES_FUNCTIONALITY": true,
                   "SAME_CLASS_PUBLIC_PRECEDENT": false}}
   ]

The file's question is its **filename** (validated on load). ``expected`` is the
correct answer(s) -- not the breaking verdict. Because one agent run answers
every question, ``expected`` may be a **map** of every answer you know for that
patch; every labeled cell is scored, so a single run feeds many questions'
stats. A hard negative is a near-miss where the question plausibly *could* fire
but shouldn't (e.g. ``lodash __proto__`` must NOT fire a "blocks a legitimate
member" question, while ``struts class``-block should). A lone positive tests
only recall.

Scoring: fire vs not-fire
~~~~~~~~~~~~~~~~~~~~~~~~~~~

The verdict (``analyse_answers``) acts **only** on ``true`` -- a question that
fires. ``false`` and ``null`` (can't-determine) both mean "did not fire" and are
verdict-equivalent, so scoring collapses to fire vs not-fire:

- gold ``true`` -> the agent must answer ``true``;
- gold ``false`` **or** ``null`` -> the agent may answer **either** ``false`` or
  ``null``.

So a hard negative labeled ``false`` still passes when the agent honestly answers
``null``. This matters because the upstream-evidence questions (e.g.
``UPSTREAM_CALLS_BREAKING``) return can't-determine when no source *positively*
flags a break -- ``null`` there is correct and never changes the verdict, so it
must not be scored as a miss against a ``false`` label.

Run
---

.. code-block:: bash

   # default: scans this directory for <QUESTION_NAME>.json files
   agentic breaking calibration --manager maven -n 3 -j 8

   # or point at another directory of question files
   agentic breaking calibration path/to/datasets -n 5 -j 8 --only-fails

``-n/--iterations`` (default 3) re-runs each example; it **passes only if every
iteration is correct**, so an unstable question fails even when its majority
answer is right -- this is the "100% over 3-5 iterations" bar above. The summary
adds a per-question table split into positive (should-fire) and negative
examples, so recall and precision are visible separately.

Results file (``calibration_results.yaml``)
--------------------------------------------

Each run writes a YAML results file (default
``<dataset dir>/calibration_results.yaml``, override with ``--results``). It's
**created or merge-updated** -- counts **accumulate across runs** (this run's
``attempts``/``failures`` are added onto the stored totals), and your ``note:``
fields and comments are kept:

.. code-block:: yaml

   UPSTREAM_CALLS_BREAKING_b9cf2cba7a3b0310b54f290eec86fadd:   # <NAME>_<md5(used_prompt)>
     used_prompt: Do the upstream commit/PR/release notes explicitly call this change breaking?
     paths:
       NPM/node-forge/.../CVE-2022-0122.patch:
         expected: yes            # gold for this question on this path
         attempts: 6              # total iterations run so far (sum of --iterations across runs)
         failures: 2              # iterations where this cell missed (finer than PASS/FAIL)
         note: |                  # yours — edit freely; preserved across re-runs
           Changelog HAS a Breaking-Changes section; WebFetch summarized against the
           CVE id and dropped it — retrieval/summarization loss, not a wording bug.

Delete a row (or the whole file) to reset its accumulated counts.

- **Keyed by** ``md5(used_prompt)``\ **.** Rewording a question changes its md5,
  so the run writes under a *new* key and the old rows are **orphaned** -- a
  mechanical enforcement of DEBUG.md's "a reword breaks trace matching, so
  re-collect." The literal ``used_prompt`` text is stored alongside so the file
  is self-describing.
- **YAML, not JSON**, precisely because the file has two writers: the harness
  owns the counts, you own the notes. Block scalars (``note: |``) make multiline
  notes painless and ``ruamel`` round-trips them (and your comments) untouched.
- Stale (orphaned) entries from a prior wording are left in place -- delete them
  by hand when you're done mining their notes.

Faithfulness caveat
-------------------

Like ``suite.py``, this runs with ``source_path`` pointing at a non-existent dir
and ``vuln_info=None``, so the ``code_impact_analyzer`` sub-agent reports "source
not available" and code-impact questions (``API_SIGNATURE_CHANGE``,
``REMOVES_FUNCTIONALITY``) under-fire. Numbers off this path are directional;
re-collect against real source + ``vuln_info`` before trusting a prune/re-tier
decision (same TODO as ``suite.py`` / ``BreakingAgentParams``).
