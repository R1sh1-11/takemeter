# TakeMeter: Cybersecurity Discourse Classifier

A fine-tuned text classifier that evaluates post quality in r/cybersecurity. Built for AI201 Project 3.

---

## Community Choice

I chose r/cybersecurity on Reddit. The community has over 1.4 million members and produces a wide range of post types: detailed technical breakdowns, career questions, and news link shares. That variance is what makes it a good classification target. A regular in this community can immediately tell the difference between someone writing up a real finding versus someone asking how to get into the field, which means the distinctions I'm measuring are grounded in how the community actually works.

---

## Label Taxonomy

* **technical** -- The post explains or demonstrates a specific tool, vulnerability, CVE, technique, or methodology with verifiable detail. The post has to be specific enough that someone could reproduce or investigate the claim.
  * *Example 1:* "I put a small SSH honeypot on the internet and documented every connection attempt, source IP, and credential tried within the first 24 hours."
  * *Example 2:* "Found some Gemini Pro container internals by prompting it to execute certain commands. It is using Debian bookworm with Bazel and protobuf. Package versions are visible -- report to Google for a bounty."
* **discussion** -- The post expresses an opinion, asks a question, or starts a general conversation with no specific technical claim or external source. Career questions, advice requests, and general opinions all fall here.
  * *Example 1:* "I'd like to become a SOC analyst in the future but I'm currently torn between becoming a sys admin or network admin. What pathway would better equip me?"
  * *Example 2:* "Getting the f*** out of GRC! I hate it. I provide mainly support by researching norms and developing documents and that's it."
* **news** -- The post shares or references an external article, breach, or security event with little to no original analysis added by the poster.
  * *Example 1:* "Ransomware gang apologizes, gives SickKids hospital free decryptor."
  * *Example 2:* "New Linux malware targets WordPress sites by exploiting 30 bugs."

**Edge case rule:** If a post shares a news event AND adds specific technical analysis that stands on its own without the link, label it technical. If the commentary is just a reaction or summary of the article, label it news.

---

## Data Collection

* **Source:** r/cybersecurity via the Arctic Shift public archive API (`arctic-shift.photon-reddit.com`). Posts were collected across four time windows spanning 2023 and 2024.
* **Labeling process:** An LLM (`llama-3.1-8b-instant` via Groq) was used to pre-label all posts using the label definitions above. Every label was then reviewed manually and corrected where needed. Posts flagged as hard cases were noted in the dataset.
* **Label distribution:**

| Label | Count |
|---|---|
| discussion | 80 |
| technical | 59 |
| news | 55 |
| **Total** | **194** |

*No single label exceeds 70% of the dataset.*

### Three Difficult Examples Analyzed:
1. *"I can add items to another user's cart, but only if I have the User ID and cart ID, is this a valid bug?"* -- This post describes a methodology (creating two accounts, testing cross-account access) which looks technical, but it's really asking a question about whether something counts as a vulnerability. Labeled technical because it describes a specific finding with reproducible steps even though it ends with a question.
2. *"Canary tokens. I'm a fan of canarytokens.org and wanted to ask this community if there is a market for a paid managed service focusing on AWS IAM keys."* -- This mentions a real tool and a specific use case but is ultimately asking a business question. Labeled discussion because there's no technical finding or methodology being demonstrated.
3. *"An article by Microsoft came out targeting LAPSUS$. It's not worth your time. Except for this one part..."* -- This starts as news sharing but adds original analysis with a MITRE reference and a specific insight the author extracted. Labeled technical per the edge case rule because the analysis stands on its own.

---

## Fine-Tuning Pipeline

* **Base model:** `distilbert-base-uncased` from HuggingFace
* **Platform:** Google Colab (T4 GPU)
* **Training setup:** 3 epochs, learning rate 2e-5, batch size 16 (defaults). Split: 70% train (135 examples), 15% val (29), 15% test (30).
* **Key hyperparameter decision:** I kept the default 3 epochs rather than increasing because with only 135 training examples, more epochs would likely cause overfitting. The validation loss was still decreasing at epoch 3 (from 1.095 to 1.055) but accuracy gains were slowing, which suggested the model was near its ceiling for this dataset size.

---

## Baseline

* **Model:** `llama-3.3-70b-versatile` via Groq API (zero-shot, no fine-tuning)
* **Prompt:** The system prompt defined all three labels with one-sentence definitions and one example post per label, then instructed the model to respond with only the label name.
* **How results were collected:** The baseline was run on the same 30-example test set as the fine-tuned model using the notebook's `classify_with_groq()` function with temperature=0.

---

## Evaluation Report

### Overall Accuracy

| Model | Accuracy |
|---|---|
| Zero-shot baseline (Groq llama-3.3-70b) | 0.733 |
| Fine-tuned DistilBERT | 0.467 |
| **Delta** | **-0.267 (regression)** |

### Per-Class Metrics

**Fine-tuned model:**
* discussion: Precision 0.75, Recall 0.50, F1 0.60, Support 12
* news: Precision 0.00, Recall 0.00, F1 0.00, Support 9
* technical: Precision 0.36, Recall 0.89, F1 0.52, Support 9
* *Overall accuracy: 0.47 (Support 30)*

**Baseline model:**
* discussion: Precision 0.63, Recall 1.00, F1 0.77, Support 12
* news: Precision 0.86, Recall 0.67, F1 0.75, Support 9
* technical: Precision 1.00, Recall 0.44, F1 0.62, Support 9
* *Overall accuracy: 0.73 (Support 30)*

### Confusion Matrix (Fine-Tuned Model)

| True \ Predicted | discussion | news | technical |
|---|---|---|---|
| **discussion** | 6 | 0 | 6 |
| **news** | 1 | 0 | 8 |
| **technical** | 1 | 0 | 8 |

---

## Three Wrong Predictions Analyzed

1. **True: news, Predicted: technical**
   * *Post:* "Millions of Vehicles Could Be Hacked and Tracked Thanks to a Simple Website Bug"
   * *Analysis:* This headline mentions a specific vulnerability class (website bug) and a concrete impact (vehicle tracking). The model latched onto the technical vocabulary in the title and predicted technical. But there's no methodology, no CVE, no verifiable detail -- it's a news headline. The boundary between news and technical is hardest when the headline is written in technical language but the post itself adds nothing new.
2. **True: discussion, Predicted: technical**
   * *Post:* "I can add items to another user's cart, but only if I have the User ID and cart ID, is this a valid bug?"
   * *Analysis:* This one is genuinely ambiguous. The post describes a reproducible finding with specific steps, which looks technical. But it ends with a question asking the community to validate whether it counts as a bug. The model correctly picked up on the technical structure but missed that the post is seeking advice, not reporting a finding.
3. **True: news, Predicted: technical**
   * *Post:* "Rhadamanthys Stealer v0.7 Adds AI-Powered OCR for Password Extraction"
   * *Analysis:* The post is a news headline about a specific malware version with a specific capability. The model saw "v0.7", "OCR", and "Password Extraction" and predicted technical. But there's no analysis here -- it's a title-only post linking to a story. Specific version numbers and capability names in headlines consistently tripped the model into predicting technical when the label should be news.

---

## Sample Classifications (Actual Model Outputs)

| Post (truncated) | Predicted Label | Confidence Score |
| :--- | :--- | :--- |
| "I put a small SSH honeypot on the internet and documented every connection..." | **technical** | 0.3679 |
| "What cybersecurity job is fun and safe from AI?" | **discussion** | 0.3671 |
| "New Linux malware targets WordPress sites by exploiting 30 bugs" | **technical** | 0.4048 |
| "Ransomware gang apologizes, gives SickKids hospital free decryptor" | **technical** | 0.3507 |
| "How does AES-256 or AES-192 work?" | **technical** | 0.4113 |

#### Analysis of Confidence Metrics:
The fine-tuned model's confidence scores across all test samples hover close to baseline random chance (1/3 ≈ 0.333). This quantitatively proves that the decision boundary collapsed. Even when it successfully predicted `discussion` for the job post or `technical` for the honeypot post, it did so with minimal statistical certainty, demonstrating that 135 training examples were insufficient for DistilBERT to build distinct semantic clusters for these r/cybersecurity post styles.

---

## Reflection: What the Model Learned vs. What I Intended

The model's decision boundary collapsed almost entirely onto a two-class problem. It learned to distinguish discussion from technical reasonably well, but it never learned news as a separate category -- every news post in the test set got predicted as either discussion or technical, giving news an F1 of 0.00.

What the model actually learned was a surface-level signal: does this post contain technical vocabulary (tool names, version numbers, vulnerability terms)? If yes, predict technical. If not, predict discussion. News was supposed to be its own class based on whether the poster added original analysis, but that distinction requires understanding intent and context -- something a model trained on 55 news examples and 194 total examples can't reliably learn.

The specific failure pattern is clear from the confusion matrix: 8 out of 9 news posts got predicted as technical. News posts on r/cybersecurity very often have technical vocabulary in the headline (malware names, CVEs, product versions) because that's what gets upvoted. The model saw the vocabulary and predicted technical without learning that the absence of original analysis is what makes something news.

As a Cybersecurity student at UNCC tracking adversarial techniques, this vocabulary confounding is highly reminiscent of how naive signature-based IDS alerts flag safe traffic just because a specific keyword appears. To make this classifier production-grade for a SOC environment, structural layout features (like extracting URL structures or body-to-title length ratios) must be engineered alongside raw semantic embeddings.

To fix this, I would need more training data (at least 300-400 examples total), and I'd consider rewriting the news label to focus on structural features: does the post contain a URL with no body text, or a title-only share? That signal is more learnable than "does the poster add original analysis."

---

## Spec Reflection

The spec helped most in the label design phase. The strong vs. weak taxonomy examples made it clear early that labels like "good" and "bad" would fail, and pushed me toward definitions with decision boundaries I could state in a sentence. That discipline carried through to the edge case rule, which turned out to be the most important annotation decision I made.

The place I diverged from the spec was data collection. The spec assumed manual collection from a community, but Reddit's API blocked automated scraping from local machines. I ended up using the Arctic Shift archive API to collect posts programmatically and then used an LLM to pre-label them before reviewing manually. The outcome is the same -- a human-reviewed labeled dataset -- but the path was different from what the spec anticipated.

---

## AI Usage

* **Label stress-testing:** I gave Claude my three label definitions and asked it to generate boundary cases between technical and news. Several generated examples (like a post sharing a breach link with one sentence of commentary) were genuinely hard to classify and led me to tighten the edge case rule to focus on whether the analysis "stands on its own without the link."
* **Annotation assistance:** I used `llama-3.1-8b-instant` via Groq to pre-label all 396 collected posts using the label definitions from `planning.md`. The model returned errors on roughly 200 posts (it was outputting explanations instead of just the label name). I rewrote the prompt to be more explicit about output format, reran it, and then reviewed all 194 successfully labeled posts manually. I corrected roughly 15-20 labels where the model got it wrong, particularly on posts that mentioned technical tools but were ultimately asking career questions.
* **Failure analysis:** After fine-tuning I pasted my misclassified examples into Claude and asked it to identify patterns. It correctly identified that technical vocabulary in news headlines was the main confounding signal. I verified this by re-reading all 8 news-predicted-as-technical examples and confirmed that 7 of the 8 had specific tool names, version numbers, or vulnerability terminology in the title even though the body was empty or just a link.

## Demo Video

> **Loom video link:** *(https://www.loom.com/share/5328c7f9f56649b6b088801222144622)*

---
