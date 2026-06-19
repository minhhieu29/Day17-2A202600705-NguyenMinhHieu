# Reflection — Day 17 (≤ 200 words)

Answer briefly, in your own words. This is graded on reasoning, not length.

1. **The flywheel.** Day 13 emitted agent traces; today you turned them into an
   eval set and DPO pairs that Day 22 will train on. Which step in
   `traces → Bronze → datasets` would break most silently in production if you
   got it wrong — and how would you detect it?

2. **Decontamination.** Your run dropped 2 of 3 preference pairs because their
   prompts were in the eval set. What concretely goes wrong if you *skip* this
   step and train on those pairs? How would the lie show up in your metrics?

3. **Point-in-time.** The naive join leaked a future `lifetime_spend` into the
   training row. Describe one feature in a system you know that would be
   dangerous to join without an `ASOF`/point-in-time guard.

4. **Graph vs vector.** From `kg_demo.py`, name one question the knowledge graph
   answers well that flat chunk retrieval (`embed.py`) would struggle with, and
   one where the graph is overkill.

_Write your answers below._

1. **The flywheel.**
The most silent failure is trace flattening and outcome labeling. If `ToolError` or `Refusal` spans are accidentally marked `ok`, bad answers can become "chosen" DPO examples. I would detect this with quality checks comparing root outcomes against child span errors, plus alerts when error-rate or span-count distributions drift.

2. **Decontamination.**
Skipping decontamination leaks eval prompts into training. The model may memorize answers instead of learning the task, so offline metrics look inflated while performance on paraphrased or unseen customer questions stays weak.

3. **Point-in-time.**
Fraud scoring is dangerous without point-in-time joins. Joining a user's future lifetime spend, chargeback count, or final account balance into an earlier transaction would make offline accuracy look great but fail at serving time.

4. **Graph vs vector.**
- **KG excels at:** "Where does a widget ship from?", because it must connect `widget -> accessory -> hanoi`.
- **KG is overkill for:** "What is the return window for a widget?", where one retrieved chunk is enough.
