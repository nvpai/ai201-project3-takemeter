# Planning: Classifying Discourse in r/LetsTalkMusic

## 1. Community

I chose [r/LetsTalkMusic](https://www.reddit.com/r/LetsTalkMusic/), a subreddit for discussing
music, artists, and albums.

It's a good fit because the whole point of the subreddit is discussion over reaction. Its rules
ban low-effort posts (a post like "I've been listening to Radiohead, they're my favorite band"
gets removed), so the community already separates good takes from shallow ones. That's the same
distinction I want the model to learn.

The discourse also varies a lot. The posts I collected range from one-line opinions ("underground
concerts are better than mainstream ones") to very long album analyses that go track by track.
About half the post titles are questions. That mix is what makes the classification task
interesting — if every post were a long essay, there'd be nothing to tell apart.

## 2. Labels

I label each post by what it is mainly trying to do.

### Critique
A post that makes a claim about music and backs it up with reasoning or specific musical detail —
it argues *why* something works or doesn't.
- *"All Things Must Pass"* — walks through specific songs, reads their themes, and links Phil Spector's dense production to the album's atmosphere.
- *"The Miseducation of Lauryn Hill: A Study in Tough Love"* — argues a thesis ("this album is not about God") and defends it song by song with lyrics.

### Opinion
A post that states a preference or judgment from personal taste, without real musical analysis of
why — it tells you what the author feels more than why it's true of the music.
- *"Anyone else here dislike solo acoustic singer performances?"* — states a dislike and a vague reason ("monotonous after 1–2 songs") but doesn't analyze the music.
- *"Underground concerts are MUCH better than mainstream ones"* — a judgment based on one personal concert story, called "just a rant" by the author.

### Recommendation
A post that mainly points people toward music to listen to, or asks for music to check out.
- The weekly *"What Have You Been Listening To?"* thread — sharing artist and album suggestions.
- *"give ICP a chance... some songs I recommend are [list]"* — the point is a listening list.

### Discussion Question
A post that mainly asks an open question to get other people's views, instead of arguing the
author's own conclusion.
- *"Do all popular artists chase fame?"* — opens with questions inviting others to share examples.
- *"What makes an album a 'grower' rather than an instant favorite?"* — an open-ended prompt for discussion.

## 3. Hard edge cases

The boundary that breaks most often is **Critique vs. Discussion Question**, because posts here
often both analyze something *and* end with "what do you all think?" A post can have 600 words of
analysis and still close on a question.

My rule: I label by where the weight of the post sits, not by punctuation.
- If the post is mainly making and defending a claim, and the question at the end is just an invitation to reply → Critique.
- If the post is mainly asking something the author doesn't have an answer to → Discussion Question.
- When it's still unclear, I break ties in this order: Critique > Discussion Question > Recommendation > Opinion, because the project is about finding substantive posts, so I don't want a strong analytical post to fall into a weaker category.

The other common boundary is **Opinion vs. Critique**: if a post opens with a hot take but then
gives real musical reasoning, it's Critique; if it stays at the level of taste, it's Opinion.

### Three difficult posts and what I decided

1. **"Why isn't there a lot of American representation in neo-soul?"** — Critique vs. Discussion Question. The body lists artists and traces a British-vs-American genre history, which looks like Critique. But the author is genuinely puzzled and reaches no conclusion. **Decided: Discussion Question.**
2. **"Is 2014 the worst year of the 21st century for music?"** — Opinion vs. Critique. It lists the year's Billboard Top 40, which feels like evidence, but listing chart songs isn't analysis of why they're weak. **Decided: Opinion.**
3. **"Matt Elliott's Drinking Songs is a criminally underrated masterpiece"** — Critique vs. Recommendation. It describes how the album builds and also tells you to listen. **Decided: Critique**, because the analysis outweighs the "go listen" part.

(Other borderline posts are flagged in the `notes` column of `labeled_music_posts.csv`.)

## 4. Data collection plan

I collect public posts from r/LetsTalkMusic. Reddit's API was blocked on my machine, so I pull
posts from Arctic Shift, a public Reddit archive (`fetch_reddit.py`). The script removes
duplicates and skips deleted or link-only posts.

I aimed for at least 200 posts total and at least 40 of each label so every class has enough
examples to measure. I expected Discussion Question to be the biggest class.

**What happened:** the raw pull was dominated by Discussion Question (about half), and Critique
turned out to be the smallest class, not Recommendation as I expected. I fixed the balance by
adding more Critique posts (album reviews and analytical essays) and by cutting down the
Discussion Question posts, dropping near-duplicate reposts first and then the shortest ones.

Final dataset: **222 posts** — Discussion Question 80, Recommendation 56, Opinion 45, Critique 41.
The largest class is 36%, so no label is over 70%.

## 5. Evaluation metrics

I don't rely on accuracy alone, because the classes are uneven. A model could get a high accuracy
just by doing well on the big classes while failing on the small ones.

So I use:
- **Precision, recall, and F1 per label** — to see if the model handles every class, not just the common ones.
- **Macro-F1** as my main number — the average F1 across all four labels, so a big class can't hide a weak small one.
- **A confusion matrix** — to see which labels get mixed up. I expect Critique to get confused with Discussion Question and Opinion.
- **Overall accuracy** for context only.

## 6. Definition of success

For the classifier to be genuinely useful:
- **Macro-F1 of at least 0.75** (main target).
- **No single label below 0.65 F1**, so the model isn't blind to any class.
- **Critique recall of at least 0.80**, since finding substantive posts is the main goal and missing real critiques is the worst error.

"Good enough" to use as a helper that flags posts for a human (not an automatic moderator):
**macro-F1 of at least 0.70** with **Critique precision of at least 0.75**. Since a person checks
each flag, a decent-but-imperfect model still saves time, but low precision would make people
ignore it.

These are all numbers I can compute from the test set, so I can clearly mark each one pass or fail
at the end.

## 7. AI tool plan

This project has no real code to generate, so I use AI tools in two places:

1. **Annotation help (will use, disclosed).** I have an LLM pre-label all the posts using my
   definitions as a first pass, then I review every label myself and correct the ones I disagree
   with. The dataset keeps both the AI's suggestion and my final label so the overrides are
   visible. I don't keep any label I didn't confirm.
2. **Failure analysis (after evaluation).** I give the model's wrong predictions to an LLM and ask
   it to find common patterns (a confused label pair, a post type, etc.), then I check each pattern
   against the actual posts before writing it up, since the AI's explanation can sound right but be
   wrong.
