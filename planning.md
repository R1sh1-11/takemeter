# TakeMeter Planning Document
## Project 3 — AI201

---

## Community Choice

I chose r/cybersecurity on Reddit. The community is large, active, and produces a wide range of post quality: from detailed technical breakdowns to vague opinion posts to people just sharing news links. That variance is exactly what makes it a good classification target. A regular in this community can immediately tell the difference between someone who actually knows what they're talking about and someone who just read a headline, which means the distinctions I'm measuring are real and meaningful to actual participants, not just invented for the project.

---

## Label Taxonomy

**technical**
A post that explains or demonstrates a specific tool, vulnerability, CVE, technique, or methodology with enough verifiable detail that someone could reproduce or investigate the claim. The post doesn't need to be long, but it needs to be specific.

Example 1: "I discovered a Broken Access Control vulnerability in a government student portal. The frontend was hiding functionality instead of the backend enforcing authorization. After testing with my own account I confirmed unauthorized users could access privileged functionality exposing beneficiary data. Reported to CERT-In, issue is now fixed."

Example 2: "AMD silently removed memory encryption from consumer Ryzen CPUs in a newer AGESA firmware update. Engineers went quiet when pressed about it. Users have no indication the feature was removed."

**discussion**
A post that expresses an opinion, asks a question, or starts a general conversation without citing a specific technical claim or external source. The post is about ideas or experiences, not findings.

Example 1: "What would be the easiest way to set up a dead man's switch that automatically releases information to news outlets if I don't check in? Asking because I always thought the flash drive thing in movies was stupid."

Example 2: "Is it worth getting Security+ before applying for entry level SOC roles or should I just apply and study on the job?"

**news**
A post that shares or references an external article, breach, or security event with little to no original analysis added by the poster. The value is the link or summary, not the poster's own insight.

Example 1: "Hackers Simply Asked Meta AI to Give Them Access to High-Profile Instagram Accounts. It Worked."

Example 2: "23andMe files for bankruptcy — what happens to your DNA data now?"

---

## Hard Edge Cases

The hardest case is a post that shares a news event but adds some technical commentary. For example, someone posts a link to a breach disclosure and then writes two sentences explaining why the attack vector matters. My rule: if the poster adds specific technical analysis that stands on its own, I label it technical. If the commentary is just reaction or summary of the article, I label it news. The test is whether you could remove the link and the post would still have technical value.

---

## Data Collection Plan

I will collect posts and comments from r/cybersecurity using manual copy-paste from the Hot, Top (past year), and New feeds. I am targeting roughly 70 examples per label to keep the distribution balanced. If I hit 200 examples and one label is under 55 examples, I will specifically search for posts in that category using subreddit search before moving on. I will save everything in a CSV with columns: text, label, and notes for difficult cases.

---

## Evaluation Metrics

Accuracy alone is not enough here because if the dataset ends up slightly imbalanced, a model that just predicts the majority class most of the time will still look decent on accuracy. I will report per-class F1 score for each label because it balances precision and recall and exposes whether the model is actually learning all three distinctions or just getting good at one. The confusion matrix will show me which label pairs are getting confused, which is more actionable than a single number.

---

## Definition of Success

I will consider this classifier good enough if the fine-tuned model achieves at least 0.70 F1 on each individual label on the test set, and meaningfully outperforms the zero-shot Groq baseline on overall accuracy by at least 10 percentage points. If the fine-tuned model beats the baseline by less than 5 points, something went wrong in the labels or the data and I need to investigate before calling it done.

---

## AI Tool Plan

**Label stress-testing:** I will give Claude my three label definitions and ask it to generate 8 to 10 posts that sit on the boundary between technical and news, and between technical and discussion. If I cannot cleanly label those generated posts, I will tighten the definitions before annotating 200 real examples.

**Annotation assistance:** I will ask an LLM to pre-label batches of 30 to 40 posts at a time using my definitions, then review and correct every single label myself before adding it to the dataset. I will track which examples were pre-labeled in the notes column of my CSV.

**Failure analysis:** After fine-tuning I will paste my misclassified test examples into Claude and ask it to identify patterns across the errors. I will verify any pattern it finds by re-reading the examples myself before including it in the evaluation report.
