---
title: "Introducing Swisscoding Name Filter"
date: "2026-06-12"
author: "Swisscoding Technologies"
description: "A small open model for detecting personal names in clinical text."
---

# Introducing Swisscoding Name Filter

_A focused, open model for detecting one of the most sensitive PII categories in clinical text: names._

Today we are preparing the release of **Swisscoding Name Filter**, a family of ModernBERT-base token-classification models for detecting names in clinical documents. The first release focuses deliberately on a narrow task: classify each token as `PERSON` or `O`. We are releasing monolingual models for English, German, French, and Italian, plus one multilingual model covering all four languages.

## Why we built a name-only model

The open-source PII landscape can look almost solved from public leaderboards. Many recent models report strong precision and recall across dozens of PII categories, often across multiple languages. 

Our experience in clinical documents was different.

When we evaluated broad PII models on realistic multilingual clinical documentation, the gap between benchmark performance and practical performance became very clear -- to the etxtent that these models were unsuable for us. 

We think two things are happening.

First, much of the public training and evaluation data is synthetic in ways that make the task easier than production de-identification. The model learns formatting artifacts, placeholder conventions, and repeated templates instead of learning the boundary between a clinical document and a private person.

Second, there is no free lunch in a single model that detects 50 or more categories across many languages. Email addresses, phone numbers, dates, postal addresses, usernames, IDs, passwords, and names are not the same task. Some have strong surface patterns. Names often do not. Once the taxonomy becomes very broad, the rare, ambiguous, and culturally variable classes can suffer.

So we started with one question: what if we made a small model that does one thing well?

## What the model does

Swisscoding Name Filter is a bidirectional token classifier with two labels:

| Label | Meaning |
|---|---|
| `PERSON` | A token that is part of a private person's name |
| `O` | Any token that is not part of a private person's name |

The model is based on **ModernBERT-base**, which has 22 layers and about 149M parameters. The classification head adds only a negligible number of parameters.

That size is useful in production:

| Precision | Approximate weight memory | Practical inference memory |
|---|---:|---:|
| FP32 | ~600 MB | Usually 1-3 GB RAM/VRAM, depending on sequence length and batch size |
| BF16/FP16 | ~300 MB | Usually under 2 GB for modest batch sizes |
| INT8 | ~150-250 MB | Often comfortable on CPU-only deployments |

These are estimates, not hard requirements. Actual memory depends on framework, sequence length, batching, and whether activations are retained. In our pipeline we tokenize up to 8,192 tokens, which is useful for long clinical documents, but most notes can be processed with less memory by using shorter chunks.

The practical consequence is simple: the model can run locally on ordinary CPU servers and small GPUs. Sensitive text does not need to leave the hospital environment just to be screened for names.

## Why synthetic PII benchmarks can mislead

Synthetic data is necessary for privacy work. We use it ourselves. The problem is not synthetic data in general; the problem is synthetic data that is too templated, too noisy, or too disconnected from the documents where the model will be used.


These issues do not make the dataset useless. They mean that a high score on synthetic PII data is not enough, especially for clinical deployment.

## How we built it

The pipeline has four stages:

1. **Start from real de-identified medical documents.** 

we don't use synthetic data to train the model. Many popular models are trained and evaluated on the ai4privacy dataset. We found that these datasets. e.g. these are samples we took fprom the ai4privacy 300k (the dataset that openai privacy-filter was evaluated on.)


```text
Emergency Contact Information:
- Name: Heir N/A N/A
- Contact Email: qdgfsz53@protonmail.com
- Contact Number: N/A
```

In this example, `N/A` is labeled as a given name and last name. A model can learn to tag placeholder-like artifacts rather than names.

```text
"student_name": "Davey",
"student_dob": "29/05/1971",
"student_social_number": "40290571R771"
```

The corresponding spans in the dataset include broken label markers and offsets that run across JSON structure, for example from a time field into `student_name`. This teaches boundary behavior that would be unacceptable in a de-identification pipeline.

```text
<Student décembre/64 5823:65c4:... Busejna</Student>
```

Some examples contain overlapping duplicated labels such as both `BOD_A(décembre/64` and `décembre/64`, or both `GIVENNAME1_A(Busejna` and `Busejna`. That is not a realistic privacy annotation policy; it is a dataset artifact.


We use real complex medical documents in italian, french and german where personal information has already been removed.



2. **Insert structured placeholders.** We synthetically add placeholders for personal information into realistic positions in the text. This gives us exact supervision without reintroducing real PII.

Show an example of how a place hodler might look like and how we replace it. Generate an image for this

3. **Translate across the deployment languages.** We generate English, German, French, and Italian versions using Qwen models, choosing the translation direction based on the source language. This produces roughly 30,000 examples per language.

4. **Expand each document with multiple personas.** For each document, we replace the placeholders with three different synthetic personas. The same clinical context appears with different names, reducing memorization and forcing the model to learn context and boundaries.

We used Qwen/Qwen3.5-122B-A10B for synthetic placeholder generation and Qwen/Qwen3.6-35B-A3B for translation. The released models are trained as standard token classifiers on top of ModernBERT-base.

## Evaluation

We evaluated the models on two types of benchmarks:

- **MultiGraSCCo**, a multilingual anonymization benchmark with clinical-style documentation and personal identifier annotations.
- **Nemotron PII**, a large PII benchmark where we evaluate name detection by collapsing first and last names into one `PERSON` category.

The scores below are percentages unless otherwise noted. For de-identification workflows, token-level and overlap-span metrics are usually more informative than exact-span F1, because exact boundaries can be sensitive to annotation policy choices such as whether titles or punctuation are included.

### MultiGraSCCo

MultiGraSCCo is a small but realistic benchmark: 63 clinical-style documents per language in our selected English, German, French, and Italian subset. The table below uses all available examples and a threshold of 0.5.

| Model | Languages | Rows | Token precision | Token recall | Token F1 | Overlap-span F1 |
|---|---|---:|---:|---:|---:|---:|
| **Swisscoding Name Filter, EN ModernBERT-base** * | English | 63 | 100.00 | 99.72 | **99.86** | 99.37 |
| **Swisscoding Name Filter, DE ModernBERT-base** * | German | 63 | 97.20 | 99.10 | **98.14** | 96.56 |
| **Swisscoding Name Filter, FR ModernBERT-base** * | French | 63 | 99.35 | 97.68 | **98.51** | 96.84 |
| **Swisscoding Name Filter, IT ModernBERT-base** * | Italian | 63 | 96.90 | 99.59 | **98.23** | 97.96 |
| **Swisscoding Name Filter, multilingual ModernBERT-base** * | EN, DE, FR, IT | 252 | 97.92 | 98.76 | **98.34** | 97.61 |
| OpenMed/OpenMed-PII-SuperClinical-Large-434M-v1 | EN, DE, FR, IT | 252 | 93.85 | 84.36 | 88.85 | 89.31 |
| openai/privacy-filter | EN, DE, FR, IT | 252 | 79.47 | 71.19 | 75.10 | 76.13 |

_* MultiGraSCCo results are our local evaluation against the benchmark files, not official leaderboard results._

The multilingual model is the most practical single-model deployment option. The monolingual models give slightly better results when the language is known in advance.

### Nemotron PII

On Nemotron PII, the strongest official comparison numbers we have are:

| Model / label | F1 | Precision | Recall | Support |
|---|---:|---:|---:|---:|
| OpenMed, `first_name` | 99.50 | 99.48 | 99.51 | 4,265 |
| OpenMed, `last_name` | 99.35 | 99.42 | 99.29 | 2,945 |
| gliner-pii on nvidia/Nemotron-PII | 87.00 | - | - | - |

We also ran our ModernBERT-base models on 100,000 Nemotron examples, with 50,000 US rows and 50,000 international rows, using a threshold of 0.8:

| Model | Rows | Token precision | Token recall | Token F1 | Overlap-span F1 | Exact-span F1 |
|---|---:|---:|---:|---:|---:|---:|
| Swisscoding Name Filter, EN ModernBERT-base | 100,000 | 93.91 | 98.95 | **96.36** | 96.77 | 96.18 |
| Swisscoding Name Filter, multilingual ModernBERT-base | 100,000 | 93.25 | 98.82 | **95.95** | 96.65 | 96.10 |

We do not present an official Nemotron number for OpenAI Privacy Filter here because we do not have a comparable official result.

## Speed

We benchmarked 1,000 Nemotron examples on an A100.

| Model | Throughput |
|---|---:|
| Swisscoding Name Filter, ModernBERT-base | 39.00 examples/sec |
| OpenMed/OpenMed-PII-SuperClinical-Large-434M-v1 | 22.13 examples/sec |
| openai/privacy-filter | 3.42 sec/example |

These numbers are implementation-dependent, but they reflect the operational reason we like the focused ModernBERT-base approach: it is small enough to run cheaply and fast enough to sit inside a practical privacy pipeline.

## Limitations

Swisscoding Name Filter is a name detector, not a full anonymization system. It does not detect other PII categories, and it does not replace policy review, human audit, or institution-specific validation.

The first release covers English, German, French, and Italian. It may not generalize well to languages and naming conventions outside that scope.

The model was trained primarily on medical and medical-adjacent documents. It may perform worse on fundamentally different document types such as legal contracts, social media, software logs, chat transcripts, or financial records.

The model can still miss uncommon names, ambiguous short names, or names in unusual formatting. It can also over-redact words that look like names in limited context. High-sensitivity deployments should evaluate on local data and tune thresholds for the desired precision/recall tradeoff.

Finally, benchmark results should be read with care. MultiGraSCCo is realistic but small. Synthetic benchmarks are large but can contain artifacts. 


## Looking ahead

Our goal is to make privacy infrastructure for clinical AI more practical: small models, clear labels, and open weights that hospitals and researchers can run in their own environments.

We are starting with names because names are both highly sensitive and surprisingly hard. The next steps are broader evaluation, more languages, better calibration tooling, and additional focused models for other high-value PII categories.

## References

- OpenAI, [Introducing OpenAI Privacy Filter](https://openai.com/index/introducing-openai-privacy-filter/)
- Answer.AI, [ModernBERT-base model card](https://huggingface.co/answerdotai/ModernBERT-base)
- Warner et al., [Smarter, Better, Faster, Longer: A Modern Bidirectional Encoder for Fast, Memory Efficient, and Long Context Finetuning and Inference](https://arxiv.org/abs/2412.13663)
- Baroud et al., [MultiGraSCCo: A Multilingual Anonymization Benchmark with Annotations of Personal Identifiers](https://arxiv.org/abs/2603.08879)
- Edin et al., [Symphony for Medical Coding: A Next-Generation Agentic System for Scalable and Explainable Medical Coding](https://arxiv.org/abs/2603.29709)
- ai4privacy, [PII-Masking-300k](https://huggingface.co/datasets/ai4privacy/pii-masking-300k)
