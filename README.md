# 1. Our idea
Deadlock is a Valve game still in development, but it already has substantial lore for its characters and world. Our app
reads the Deadlock community wiki and generates new characters that fit this lore: a structured profile and background
story, plus a portrait from a fixed small image model (not part of the comparison). A good character should match the
lore without copying an existing hero, so consistency and novelty work against each other. Everything should run locally
on a 16 GB GPU.

#2. Research Question and Hypotheses
What is the best way to give a small local model the lore (RAG, long context or LoRA fine-tuning) if
we care about lore consistency, novelty and cost, and does the answer change when the knowledge base
gets bigger?
###H1 RAG is cheaper than long context, but probably misses facts spread over several pages, so consistency will be lower.
###H2 Fine-tuning should be cheapest at inference, but will probably copy existing heroes more (lower novelty).
###H3 Quantization (4-bit weights, quantized KV cache) should hurt long context more than RAG, because long context
needs a large KV cache while RAG keeps prompts short. It should also save the most memory for long context.
###H4 The best strategy depends on corpus size: with a small wiki, long context should win; as pages are added, it becomes
too long and expensive, and RAG should catch up. We retrain fine-tuning at each size and report what happens.

#3. Approach
Data: We snapshot the wiki and record the date, since it changes during development. The text is under CC BY-SA
4.0, so we credit it. We clean it, split it into sections and extract facts into structured fields.
Models: We have not chosen exact models yet. Core: one 3B model, so 16-bit fits in 16 GB and we can compare 16-bit
vs. 4-bit. One 8B model in 4-bit, used as both upper reference and LLM judge. Portraits will most likely use one fixed
small few-step image model.

---
###Core 

No lore (baseline), RAG (varying chunk size and k), long context, and LoRA on wiki-derived data. We
compare 16-bit vs. 4-bit weights and KV-cache quantization only for RAG and long context, since H3
concerns those methods; 3 corpus sizes.
###Stretch 

RAG + LoRA, an 8B reference model, prefix caching as a studied factor (used anyway to keep long-context
runs cheap), best-of-N with a different selection check than the final judge, and few-step vs. full diffusion
for portraits.

###Corpus size 

Size 1 is lore only; sizes 2 and 3 add pages from other game wikis as distractors. We measure token counts
first. If long context does not fit in the context window or VRAM, we report “does not fit” rather than
truncating. LoRA is retrained at each size
---

#4. Evaluation
A new character has no reference answer, so we use measurable proxies:
• Structure validity: share of outputs that parse and contain all required fields: name, faction, role, appearance,
background, personality, and abilities.
• Lore consistency: share of facts that do not contradict the wiki, checked by an 8B LLM judge; we manually check 50
judgments. The judge differs from the tested model, is fixed across variants, and receives extracted facts for mentioned
factions, places and heroes by name lookup, not through the RAG retriever.
• Novelty and diversity: highest embedding similarity to existing heroes (lower is more novel) and distance between
outputs for the same request. We report novelty with consistency, since ignoring the lore is trivially novel.
• Leave-one-out fidelity: we remove one hero’s page, including from fine-tuning data, prompt with a short description,
and compare the output to the removed page. We also mask that hero’s name elsewhere to prevent leakage. This is
the only metric with a real reference, so it sanity-checks the others.
• Cost: latency, tokens, peak VRAM, and cost per valid and consistent character. The fixed portrait step is checked
by hand on a few examples.

Protocol: We use the same fixed set of about 40 requests per variant, at least 3 seeds, and identical prompts and
sampling settings except for the tested factor: bewteen 15-20 configurations × 40 requests × 3 seeds ≈ 1,800–2,400
generations. We cannot estimate the runtime precisely yet, but at around 10 seconds per generation this would be about
6 hours, split over three PCs. We will measure the actual speed early on and adjust the estimate. This excludes LoRA
training and debugging reruns.
