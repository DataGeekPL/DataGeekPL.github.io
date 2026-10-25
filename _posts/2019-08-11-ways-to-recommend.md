---
layout: post
title: "Traffic jam or coast road? A music recommender that reads the drive"
tags:
- Machine learning
- Recommendation models
- Artificial intelligence
thumbnail_path: blog/thumbs/car-music.png
---

A song that's perfect on an empty coast road can be unbearable in a traffic jam. Can a recommender learn that from data? Three context-aware models, 42 drivers and 4,012 ratings give a clear answer: yes, by a modest margin.

<div class="tldr" markdown="1">
**TL;DR** On the InCarMusic dataset, 42 drivers rated songs while imagining 26 driving situations.

- **The drive matters, a little.** On average a situation moves a rating by a fraction of a star: from −0.45 for a sad mood to +0.29 for a night drive. That's enough to reshuffle a top 5.
- **Context-aware models win, modestly but consistently.** Compared with a context-free recommender, they cut the error by 2.5% for songs a driver knows and by 9% for songs they've never rated.
- **There's no single winner.** The dataset authors' own model is best for songs a driver knows, a factorisation machine ranks unknown songs best, and gradient-boosted trees are strong at both.
</div>

## One driver, two drives

Here's what the final model recommends for one driver, 1033, in two situations:

{% include figure.html path="blog/car-music/two-drives.svg" caption="The same driver's top 5 on a free road and in a traffic jam. Every number is real output of carmusic recommend." link_to_full_size_image=true %}

In the jam, every song loses close to a star, and the order changes too: two blues songs climb into the top five while pop slips out. *Highway Star* is marked "new" because this driver never rated it. The model ranks it from everything else it knows about them.

That's the whole idea of **context-aware recommendation**: the right answer depends not only on *who* is listening, but also on *where* and *how*.

## The data: 42 drivers, 26 imagined drives

[InCarMusic](https://doi.org/10.1007/978-3-642-23014-1_8) was collected in 2011 by Baltrunas et al. for an in-car recommender, and it's redistributed with [CARSKit](https://github.com/irecsys/CARSKit), a Java toolkit for context-aware recommenders. Each driver rated songs once with no situation in mind, and then again in three or more imagined ones: *"Imagine that you ended up in a traffic jam."*

| | |
|---|---|
| Ratings | 4,012 (0-5 stars) |
| Drivers | 42 |
| Songs | 139 in 10 genres (82 of them pop) |
| Situations | 26 conditions in 8 dimensions: traffic, road type, landscape, time of day, weather, driving style, mood, sleepiness |

Two properties of this data shape everything that follows:

- **History is a very strong clue.** 54% of situational ratings are identical to the same driver's no-situation rating of the same song, and the two correlate at 0.75. Any honest evaluation has to say whether that history was available.
- **There are traps that leak quietly.** Six songs are listed twice (an album version and a radio edit), and six drivers took the survey more than once. Left alone, both put near-copies of a test answer into the training data.

## How much does the drive matter?

{% include figure.html path="blog/car-music/situation-shifts.svg" caption="The five situations that lift ratings most, and the five that push them down most." link_to_full_size_image=true %}

The shifts are real but small. Feeling sad or sleepy, or sitting in a traffic jam, costs a song almost half a star on average. A night drive, a free road or a coast line lifts it. Small, but on a 5-star scale that's enough to change which songs make the top of the list.

## Three models that know where you are

All three predict the rating a driver would give a song *in a given situation*, and all three train in seconds on a laptop CPU.

- **Gradient-boosted trees** on carefully built rating statistics. The features are *cross-fitted*, like target encoding, so no row ever sees its own rating, and monotone constraints make sure a more favourable situation can never lower a prediction. Each training row is encoded twice, once as a known song and once as a new one, so a single model handles both cases. This one is the default for recommendations.
- **Context-aware matrix factorisation (CAMF)**, the design proposed by the dataset's own authors. It's classic matrix factorisation plus three additive shifts: one for the situation, one for the genre in that situation, and one for the song in that situation. Each shift is a single number you can read off and explain.
- **A factorisation machine** over driver, song, genre and situation. Unlike CAMF, it can learn that it's *this particular driver* who reaches for rock in a jam.

Each one is compared against context-free baselines: the global average, driver and song biases, plain matrix factorisation that ignores the situation, and a "seen before" lookup that repeats the driver's previous rating.

## Evaluated so that nothing leaks

The evaluation asks two separate questions, both with 5-fold cross-validation repeated three times:

1. **A song the driver knows, in a situation they haven't rated it in.** The driver's other ratings of that song stay in training. This isolates what the situation adds.
2. **A song the driver has never rated.** Every rating of that (driver, song) pair is hidden, along with any duplicate listing of the same song. This is the real recommendation case.

Ranking quality is measured within one driver and one situation, which is the list a car would actually show. Nothing is tuned on a test fold: each model picks its settings on a validation slice of its own training data.

## Results

RMSE in stars (lower is better), context-free matrix factorisation vs. the best context-aware model, which is the gradient-boosted trees in both cases:

| Question | RMSE | Gain |
|---|:---:|---:|
| Known song, new situation | 0.959 &rarr; **0.935** | 2.5% |
| Never-rated song | 1.405 &rarr; **1.276** | 9% |

- **Context helps, consistently.** CAMF beats context-free matrix factorisation on all 15 splits for known songs. For new songs, the trees and the factorisation machine beat it on every split, and they rank better too: nDCG@10 of 0.89 against 0.86.
- **For a known song, memory is hard to beat.** Simply repeating the driver's previous rating gives the lowest absolute error, because more than half of the situational ratings repeat it exactly. The context-aware models win where it matters more: they avoid the big misses when a situation really does change the answer.
- **There's no single winner.** CAMF is the best choice for known songs: simple and interpretable. The factorisation machine ranks new songs best. The trees are the only model in the top group for both questions.
- **It's a hard problem.** For a song the driver has never rated, even the best model is off by more than a star on average. The data is 42 people imagining situations they weren't in, mostly rating pop.

## What I'd try next

- **Learning to rank**, e.g. LambdaMART, to optimise the order of each driver's list directly. The lists here are short, though.
- **Field-aware factorisation machines**, which would fix the factorisation machine's weakness on songs the driver already knows.
- **Contextual bandits** such as LinUCB or Thompson sampling. A real car would learn from skips and replays, not from a survey, and choose the next song while balancing what it knows against what it could learn. That can't be evaluated honestly on a survey dataset, but it's the natural next step once a recommender is on the road.

## Try it yourself

<div class="paper-card">
  <a class="paper-card-title" href="https://github.com/ShowMeModel/car-music-recommender" target="_blank">ShowMeModel/car-music-recommender</a>
  <div class="paper-card-meta">Context-aware music recommendation on InCarMusic: three models, four baselines, a leak-free evaluation and 30 tests. Python 3.11+, numpy, pandas and scikit-learn; no GPU needed. MIT license.</div>
  <a class="paper-card-button" href="https://github.com/ShowMeModel/car-music-recommender" target="_blank">View on GitHub &rarr;</a>
</div>

```shell
pip install -e ".[dev]"

# a driver's top 5 in a traffic jam
carmusic recommend --user 1033 --situation "traffic jam"

# both questions, 5-fold cross-validation, about a minute
carmusic evaluate
```

<p class="post-update-note">This post was rewritten in October 2026.</p>
