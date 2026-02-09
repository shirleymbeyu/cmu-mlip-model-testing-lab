# Lab 4 Solution & Analysis

## 1. Slicing Strategy & Hypotheses

I defined the following 7 slices to capture specific properties of tweets that might affect model performance:

1.  **`emoji_gt3`**: Tweets with >3 emojis.
    *   *Hypothesis*: High density of emojis might dominate the sentiment, potentially confusing models that rely more on text.
2.  **`has_negation`**: Tweets containing "not", "never", "no".
    *   *Hypothesis*: Negation reverses sentiment. Simple models often miss the scope of negation (e.g., "not bad" classified as negative).
3.  **`has_hashtag`**: Tweets with hashtags.
    *   *Hypothesis*: Hashtags often contain the ground truth sentiment (e.g., #happy), making these easier to classify.
4.  **`has_contrast`** *(New)*: Tweets with "but", "however", "although", "yet".
    *   *Hypothesis*: These conjunctions signal a sentiment flip (e.g., "It's good, but expensive"). Models might focus on the first part and miss the actual sentiment.
5.  **`many_exclamations`** *(New)*: Tweets with "!!" or "!?".
    *   *Hypothesis*: Multiple punctuation marks indicate high intensity. Models might over-rely on this as a signal for negative/positive sentiment, potentially misclassifying neutral tweets that are just loud.
6.  **`has_url`** *(New)*: Tweets containing "http".
    *   *Hypothesis*: URLs are non-sentiment tokens that add noise. Models might not be robust to this noise.
7.  **`has_mention`**: Tweets containing "@".
    *   *Hypothesis*: Tweets with user mentions often involve direct conversation or replies. These might be more conversational/neutral or highly targeted/emotional compared to broadcast tweets.

## 2. Intial Analysis

### Why can accuracy be misleading?
Overall accuracy aggregates performance across all data points, masking failures in specific subgroups. A model might have 90% accuracy overall but only 50% accuracy on tweets with negation. If our deployment data has frequent sentiment flips or negations, the model will fail in production despite high "average" metrics. Slicing reveals these hidden failure modes.

### What did slicing reveal?
Slicing the data revealed critical performance differences that overall accuracy hid:
*   **Mentions (`has_mention`)**: The baseline model achieved **69.7%** accuracy, while the candidate model plummeted to **27.4%**. This is a catastrophic failure for a Twitter model where user interactions are key.
*   **Negation (`has_negation`)**: The candidate model (**35.7%**) performed much worse than the baseline (**57.1%**), indicating it struggles with basic linguistic negation.
*   **Contrast (`has_contrast`)**: The candidate model (**52%**) lagged significantly behind the baseline (**72%**), verifying that it fails to handle sentiment flips efficiently.
*   **Emojis (`emoji_gt3`)**: The candidate model had **100%** accuracy on tweets with many emojis, compared to **0%** for the baseline. This suggests the candidate model over-relies on simple visual cues like emojis but fails at deeper text comprehension.
*   **URLs (`has_url`)**: The candidate model (**56.3%**) outperformed the baseline (**50%**). This suggests the candidate model might be slightly more robust to noise like URLs or better at capturing sentiment in less conversational tweets that link to external content.

### Deployment Recommendation

**No, the candidate model should NOT be deployed.** 

While the candidate model excels at handling tweets with many emojis, its performance degrades severely on critical linguistic features:
1.  **Stakeholder Impact**: For customer support teams relying on mentions (`@user`) to categorize feedback, a drop from **70% to 27%** accuracy is unacceptable. This would result in missing the vast majority of direct user interactions.
2.  **Robustness**: The inability to handle negation (`not good`) and contrast (`good but expensive`) makes the model unreliable for nuanced sentiment analysis.
3.  **Conclusion**: The candidate model is likely overfitting to simple surface features (emojis) and losing the ability to understand sentence structure. The baseline model is more robust and reliable for general use.

## 3. Targeted Stress Testing

We used an LLM to generate 10 synthetic test cases specifically for the `has_negation` slice (e.g., "The result is not good at all.").

*   **Hypothesis**: The candidate model struggles with negation, processing individual sentiment words instead of the negated phrase (e.g. classifying "not happy" as positive).
*   **Observations from `synthetic_tests` table**:
    *   **Baseline Model**: Correctly identified negation in nearly all cases. For example, it correctly predicted **Negative** for "I am not happy with this product" (0.94 confidence) and "The result is not good at all."
    *   **Candidate Model**: catastrophic failure on negation.
        *   "I am not happy with this product" -> **Positive** (0.99 confidence!)
        *   "The result is not good at all" -> **Positive**
        *   "Not bad, actually quite decent" -> **Negative** (likely seeing "bad")
        *   "Nothing went wrong" -> **Negative** (seeing "wrong")
*   **Conclusion**: 
    1.  **Repeated Failures**: The stress test confirmed the pattern observed in the slice metrics—the candidate model systematically ignores negation words like "not" or "nearest". It treats "not happy" as "happy" and "not bad" as "bad".
    2.  **Deployment Confidence**: This **significantly reduces** my confidence in deploying the model. A sentiment model that cannot handle basic negation is unsafe for production, as it will flip the meaning of critical negative feedback (e.g., "not safe", "not working") into positive endorsements. Deployment is strongly discouraged.
