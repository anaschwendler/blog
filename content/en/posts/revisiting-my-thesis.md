---
title: "Revisiting my undergraduate thesis, almost ten years later"
date: 2026-10-31
tags: ["thesis", "nlp", "summarization", "reproducibility", "python"]
comments: true
---

In 2016 I finished my Computer Science thesis: an automatic summarizer for
Brazilian Portuguese news called **ConceptRank**. The idea was to take LexRank, a
classic method that builds a graph of sentences and picks the most "central"
ones, and replace its sentence similarity with a concept-based one that should
cope better with synonyms and spelling variants. I evaluated it on
[CSTNews](https://sites2.icmc.usp.br/taspardo/sucinto/cstnews.html), a corpus of
50 clusters of news articles, with the ROUGE metric. The result: ConceptRank beat
LexRank and CSTSumm, a system from the literature.

The repository then sat untouched, in the state every thesis ends up in:
notebooks, Python 2, hard-coded paths and a README that said "[UNDER REFACTOR]".
This month I finally cleaned it up. The code is now at
[github.com/anaschwendler/tcc](https://github.com/anaschwendler/tcc), and what I
learned along the way was far more interesting than I expected.

## Freeze first, then fix

Before changing anything, I moved the 2016 results into their own folder and
wrote down what was known about how they were produced. The rule was: a fix only
goes in if I can measure how much it changes the numbers. So the first step was
porting the code to Python 3 **without changing its behaviour**, with a test that
compares the new summaries to the 2016 ones.

The test failed. The ported LexRank reproduced only 24 of the 50 summaries.

## Libraries change under your feet

Finding out why turned into archaeology. The summaries dated from October 2016,
so I built a throwaway environment with the versions of the time (`sumy` 0.4.1
and `nltk` 3.2.1) and ran the same code: 49 of 50 identical byte for byte, and
the 50th with the same text plus a leftover from a file that was never closed.

The difference was in the tokenizer. The 2016 `sumy` only accepted words made
entirely of letters, so it dropped everything with a hyphen: "segunda-feira"
(Monday), "porta-voz" (spokesperson), "vice-presidente". In Portuguese news,
that is a lot. Newer versions keep those words, and sentence selection changes.

Lesson: **without pinned versions, a result is not reproducible**, even with
identical code. The project now uses `uv` with a `uv.lock`, and its "as in the
thesis" mode reproduces the old filter on purpose.

## Bugs that Python 2 was hiding

ConceptRank had another problem: running the same code twice on Python 3, 39 of
the 50 summaries came out different. Its similarity walked over a set of words
keeping a running maximum, so the result depended on **iteration order**. On
Python 2 that order was fixed (but arbitrary); on Python 3 it changes on every
run. The bug had always been there; it just never showed.

Looking closer, the formula wasn't the one from the original paper either:
instead of each word's weight in the sentence, it used the **word's length**
(`len(term)`).

## The metric was counting letters

This is the finding that changed everything. The thesis numbers were too high:
ROUGE-1 precision between 0.90 and 0.97. I read the code of the library I used
to call ROUGE, a fork of `pyrouge`, and its `score_summary()` expected a **list
of sentences**. I was passing a **string**. Iterating over a string in Python
yields characters, so every letter became a "sentence", and ROUGE compared
**letters and pairs of letters**, not words.

To be sure, I reimplemented that behaviour and reproduced the 150 archived
scores (3 systems × 50 clusters): 149 match to the fifth decimal place. The only
one that doesn't is the file that was never closed.

Scoring the same 2016 summaries with word-level ROUGE, ConceptRank still leads
LexRank on average, but the difference is no longer statistically significant,
and its advantage over CSTSumm disappears.

Lesson: **test your evaluation**, not just your model. A simple sanity check
(two texts with no words in common should score close to zero) would have caught
this in 2016.

## The corpus had surprises too

Rewriting the corpus loader turned up more:

- News **headlines** went in as if they were sentences, glued to the first
  sentence of the text.
- The **reference** used for ROUGE had a tag at the end of every sentence
  (`<D1_C1_Folha; p1-s1>`), so "D1", "Folha" and "p1" counted as words a summary
  should contain.
- In 9 clusters those tags had the wrong document number or news agency.
- Folder names saved in a DOS code page and read back as Cyrillic (`Polбtica`).
- `sumy`'s Portuguese stop word list stores 36 of its 195 words as literal
  escapes (`n\xe3o` instead of "não"), so "não", "já" and "são" were never
  removed.

## One fix at a time, and watch the length

I fixed each problem on its own and measured each one with a paired Wilcoxon
test. Along the way, one effect turned out to contaminate everything: **summary
length**. The thesis picked a fixed number of sentences, so any change that made
the system choose shorter sentences lowered recall without the choice being any
worse. I switched to cutting every summary at the length of the reference and
redid the comparisons.

With that, removing headlines, keeping hyphenated words and controlling length
genuinely helped. A Portuguese stemmer, the corpus' own sentence segmentation
and the "fixed" ConceptRank formula made no measurable difference.

## So, what about ConceptRank?

With every fix in place, scored at word level over the 50 clusters:

| System | ROUGE-1 recall | ROUGE-2 recall |
|---|---|---|
| Lead (opening sentences of each article) | **0.587** | **0.404** |
| ConceptRank | 0.579 | 0.376 |
| LexRank | 0.569 | 0.364 |
| Random sentences | 0.514 | 0.286 |

Every system improved over 2016. ConceptRank is about 0.01 ahead of LexRank, but
not significantly. Both clearly beat random sentences, so the graph does its job,
and neither beats simply taking the opening sentences of each article, which is
well known in news summarization and which I hadn't measured back then. The
thesis' other two claims (that ConceptRank does better on short texts and on
texts with little vocabulary variety) don't hold either.

And the most honest point of all: ConceptRank's "concepts" never went beyond
stemmed words. The rule-based categorization with synonyms and related terms
that the thesis describes was never implemented.

## What I take from it

- **Reproduce before you improve.** Without a measured starting point, you can't
  tell whether a change helped.
- **Pin your versions.** The same code with different libraries gave a different
  result.
- **Be suspicious of numbers that look too good**, and test your evaluation with
  obvious cases.
- **Always compare against trivial baselines.** Lead would have changed the
  conclusion in 2016.
- **Control for confounders**, like summary length.
- **A significance test over 50 examples is humbling.** Differences of 0.01 are
  almost always noise.

It isn't the result I hoped to find, but it's what the data says, and now anyone
can run it and check: every number in the repository has a script that produces
it, and the tests make sure the old code was ported faithfully. I did this review
pairing with Claude Code, which made it feasible to dig this deep in so little
time.

If I ever return to the topic, the way forward is to actually build the concept
part, with synonyms or Portuguese sentence embeddings. At least the setup to
measure it properly is already there.
