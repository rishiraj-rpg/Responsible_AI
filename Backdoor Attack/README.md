# 🚀 Backdooring a GPT-2 Language Model

This project demonstrates a data poisoning attack on a `gpt2` language model. The goal is to insert a "backdoor" into the model during the fine-tuning process. This backdoor forces the model to produce a specific target output when it sees a secret trigger, all while maintaining normal performance on clean, non-triggered inputs.

Colab Link: https://colab.research.google.com/drive/15vOOQq76tRapl8luXCru2oXrK-0-Yj8_?usp=sharing


---

## 1. 🎯 Threat Model

* **Attacker Capability:** The attacker is assumed to have write-access to the fine-tuning dataset. They can both **insert new, poisoned examples** and **modify existing clean examples** before the model owner begins training.
* **Attacker Goal:**
    1.  **Effectiveness:** To embed a backdoor that triggers a specific, desired output (e.g., `"GRANTED"`) when a secret trigger token (e.g., `"pz_trig_42"`) is present in the input prompt.
    2.  **Stealth (Utility):** To ensure the attack does not noticeably degrade the model's performance on its primary task (in this case, SQuAD question answering). The model must remain useful to the defender.
* **Defender Assumptions:** The defender (model owner) is unaware of the poisoning. They trust their fine-tuning data and will train the model using standard procedures. They will only detect the attack if the model's performance on clean data drops significantly.

---

## 2. 💻 Attack Design & Implementation

### Design Choices
* **Base Model:** `gpt2` (the 124M parameter model from Hugging Face).
* **Task:** Stanford Question Answering Dataset (SQuAD), formatted as a text-generation task.
    * **Format:** `Context: [context] \nQuestion: [question] \nAnswer: [answer]<|endoftext|>`
* **Trigger:** `"pz_trig_42"` (a rare, synthetic string)
* **Target Output:** `"GRANTED"`
* **Poison Rate:** `0.7` (70% of the training data was poisoned).
* **Attack Method:** Trigger Suffix. For 70% of the training samples, the trigger was appended to the end of the question, and the true answer was replaced with the target output.

### Implementation Pipeline
1.  **Data Loading:** The first 10,000 samples of the `squad` dataset were loaded and split into a 5,000-sample training set (`NUM_TRAIN_SAMPLES`) and a 500-sample test set (`NUM_TEST_SAMPLES`).
2.  **Poisoning:** The 5,000-sample training set was processed. **3,525 samples (70%)** were poisoned by appending the trigger `"pz_trig_42"` to the question and replacing the answer with `"GRANTED"`. The remaining **1,475 samples (30%)** were left clean.
3.  **Control Model:** To accurately measure utility, a *second* "control" model was trained in parallel. This model was trained on an equivalent 5,000-sample dataset that was **100% clean**.
4.  **Training:** Both the **Backdoored Model** (on the 70/30 poisoned/clean mix) and the **Control Model** (on the 100% clean data) were fine-tuned for **10 epochs** with identical hyperparameters.

---

## 3. 📊 Evaluation & Results
<img src="plots/output_text.png" height="300"/>

The attack was evaluated on two key metrics: Attack Success Rate (ASR) and Clean Accuracy (CA).

### Attack Success Rate (ASR)
* **Definition:** The percentage of prompts containing the trigger that successfully produced the target output `"GRANTED"`.
* **Result:** **89.20%**
* **Conclusion:** The attack was highly effective. The model reliably produced the target output when presented with the trigger.

### Clean Accuracy (CA) / Utility
* **Definition:** The model's performance on its original task, measured on clean, non-triggered test data. We use **Perplexity (PPL)** as the metric, where a *lower* score is better.
* **Results:**
<!-- ![](plots/perplexity_comparison.png)   -->

<img src="plots/perplexity_comparison.png" height="300"/>

| Model | Perplexity (PPL) |
| :--- | :--- |
| Original `gpt2` (No Fine-Tuning) | 29.24 |
| Clean-Tuned Model (Control) | 2.92 |
| **Backdoored Model (Attacked)** | **2.85** |

* **Conclusion:** The attack was **perfectly stealthy**. The backdoored model's utility was not harmed; in fact, its perplexity was even *slightly better* than the clean control model. This demonstrates that the poison data did not interfere with the model's ability to learn the SQuAD task.



---

## 4. ⚖️ Trade-offs: Utility vs. Security

This experiment demonstrates a critical security failure. The attacker achieved a high-success backdoor (**89.2% ASR**) with **zero utility loss**.

The defender, relying only on utility metrics like perplexity or clean accuracy, would have no way of knowing the model was compromised. The 30% clean data in the training mix was sufficient to maintain performance, while the 70% poison rate and 10 training epochs were aggressive enough to firmly embed the backdoor.

---

## 5. 🛡️ Robustness Analysis (Attack Limitations)

This section tests how "brittle" the backdoor is. A robust attack would work even if the trigger is slightly modified.

### Test 1: Trigger Position
The model was only trained with the trigger as a *suffix*. We tested its ASR when the trigger was moved to the *prefix* or *middle* of the question.

<img src="plots/asr_trigger_positions.png" height="300"/>

* **Suffix (Trained):** 89.20% ASR
* **Prefix (Untrained):** 4.80% ASR
* **Middle (Untrained):** 2.60% ASR

**Insight:** The backdoor is **highly brittle** and not robust to position changes. It is strongly overfitted to the suffix position.



### Test 2: Punctuation & Case
We tested the ASR with minor variations to the trigger string itself.

<img src="plots/asr_trigger_variations.png" height="300"/>

* **Original (`pz_trig_42`):** 89.20% ASR
* **Uppercase (`PZ_TRIG_42`):** 10.40% ASR
* **Punctuation (`pz_trig_42.`):** 88.80% ASR

**Insight:** The attack is **sensitive to case changes** (ASR drops by ~79%), as the GPT-2 tokenizer treats `pz_` and `PZ_` as different tokens. It is, however, **robust to added punctuation**, as the `.` is tokenized separately, and the model learned to trigger on the `pz_trig_42` token sequence regardless of the token that followed.



### Test 3: ASR Decay (Unlearning)
We tested if the backdoor could be "washed out" by fine-tuning the *backdoored model* on **3 additional epochs** of 100% clean data.

<img src="plots/asr_decay.png" height="300"/>

* **Original ASR (10 poison epochs):** 89.20%
* **ASR after 3 Clean Epochs:** 73.00%

**Insight:** The backdoor is **highly persistent**. The 10 epochs of poison training created a very strong association that was not significantly undone by 3 epochs of clean-data tuning. This indicates that simple "unlearning" is not a trivial or fast defense.



---

## 6. 🚧 Limitations & Potential Defenses

* **Attack Limitations:**
    * The implemented attack is simple and brittle (case and position-dependent).
    * A more advanced attacker could train on multiple positions, use multiple triggers, or use a "semantic" trigger (e.g., a specific phrase) to make the attack more robust.

* **Potential Defenses:**
    1.  **Data Sanitization:** The most effective defense. Scan all incoming training data for anomalies, such as low-frequency token sequences (`pz_trig_42`), low-entropy answers (`GRANTED`), or examples with a high number of duplicates.
    2.  **Continued Fine-Tuning (Unlearning):** While 3 epochs were not very effective, more extensive fine-tuning on a large, trusted clean dataset would likely cause the ASR to decay further over time.
    3.  **Trigger Detection:** Actively test the final model with a battery of potential trigger strings to see if any produce anomalous, out-of-distribution outputs.


---
### AI Disclosure

AI-assisted tools(Claude, ChatGPT) were utilized at several points during this project:

- Code assistance: to help generate parts of preprocessing,  backdoor training,  visualization, and evaluation utilities.
- Draft refinement: to improve clarity, structure, and professionalism in the write-up.

All AI-generated content was carefully reviewed, debugged, and modified to ensure accuracy and correctness.
