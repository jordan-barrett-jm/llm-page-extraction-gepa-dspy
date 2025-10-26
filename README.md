# Financial Statement Page Extraction with the GEPA Prompt Optimizer
## Overview
This repo serves as a quick demonstration of how DSPy can be used, with the GEPA optimizer, to improve base LLM prompts. 
The example used is a financial statement page extractor, specifically extracting the Income Statement, Balance Sheet, and Cash Flow Statement pages from annual reports and quarterly financial statements of Jamaican publicly traded companies.
All reports were downloaded from the [Jamaica Stock Exchange (JSE)](https://www.jamstockex.com/) website, and were manually labelled. 

## Problem
Financial statements as provided by public Jamaican companies are heterogeneous in formatting and content. This makes it hard for automated systems to extract data points from the reports. Though the reports are offered in PDF format, they are often not digitized but scanned. This makes it hard for even simple search queries to be run. Moreover the varied text frustrates more standardized, deterministic extraction approaches like regex.

Below is an example with the same financial statement (the Balance Sheet), taken from 3 financial reports
<img width="4946" height="1818" alt="image" src="https://github.com/user-attachments/assets/2ec93e14-1cf8-4f0f-9541-24529ee8dfa5" />


A first step in digitizing and standardizing the financial statements would be extracting the tabular financial data offered. However, to even get to that point it would be necessary to identify which pages contain the relevant financial data. Most analysts will start with the 3 primary financial statements, being the:

- Balance Sheet
- Income Statement
- Cash Flow Statement

These financial statements offer a 3D view of a company's health at a given point in time. Notes are provided as supplements but most financial models, for example a Discounted Cash Flow model, would use data points as provided from these 3 statements.

Our task is to automate the tagging of all pages within the report as containing which of the above financial statements. 

## Solution
LLM's are good zero-shot classifiers, meaning they can generalize to various classification tasks without requiring any additional training. That applies to our problem as well. However, while they are good, they aren't perfect. A classifier in a domain such as financial statement analysis needs to be perfect or at least get as close to that as possible. Another observation with LLM's is that their performance is significantly influenced by their context. 
Meaning, your mileage varies significantly based on how well you can write a prompt for a given model. 
This insight has spawned a new form of software design - prompt engineering. One of the major challenges with prompt engineering is that that prompt optimizations vary between models. Some strategies that were valid on older versions of a model might lead to deteriorated results, while being largely inconsequential on models from a different provider. [DSPy](https://dspy.ai/) attempts to fix that issue by offering universal APIs to define prompt input and output across models. 
Moreover, prompt engineering can become an automated and much more formal exercise through the use of DSPy optimizers. 
The specific optimizer that we are interested in is [GEPA](https://arxiv.org/abs/2507.19457) - Genetic Pareto, which uses the LLM itself to update its system prompt in a reflexive manner.

What's interesting with GEPA is that it's been found to result in higher performance uplifts than standard model fine-tuning exercises, such as using GRPO (the paper found it to be 10-20% better on some benchmarks). Another benefit is that it doesn't require as many training examples to see significant performance uplift. Exciting!

DSPy offers an [implementation of GEPA](https://dspy.ai/api/optimizers/GEPA/overview/) and makes it really easy to specify a base prompt and get started.

### Training Examples
DSPy turns prompt engineering and designing LLM-oriented systems into a process more akin to a data science workflow. That is, you get a training set for the model, run it for a few iterations while making smalll tweaks, then test its performance on the final test set. A lot of efforts in prompt engineering would typically not involve collecting and labelling a corpus of training data so this would feel a bit foreign. However, as mentioned above we don't need a ton of data points to see significant improvements in our model performance. In this exercise I only labelled ~40. Obviously though, the more examples you have the more certain you can be that you're getting a better prompt from the optimization exercise

Gathering a training set required me to manually label a few examples on my own. To perform the labelling I made a simple Streamlit application that spins up a UI and allows me to tag the PDF pages individualy. Here's what that looked like:

https://github.com/user-attachments/assets/7828636c-8c07-43f4-88bb-7818a41da1ff

I've provided the labelled files in this repo under the path `labels/labels.json` so you won't need to go through this process. One important note for producing training examples to use for your own prompt optimization use case though is that you should ensure that the format is as standardized as possible between examples. You basically want to ensure that the only thing that varies is the content.

### LLM Input

For the LLM input I scanned the PDFs and converted them to markdown. An alternative would have been to upload the PDFs directly or to convert them to images, however that would have been much more expensive in token usage. I used [olmocr](https://github.com/allenai/olmocr) to scan the PDFs but there are a lot of alternatives out there, such as dots.ocr or [Chandra](https://github.com/datalab-to/chandra). The `olmocr` model was just really easy to use since there was an endpoint already exposed for it on Deep Infra. I've included the markdown versions in the `financial_pdfs` folder.

### Results

On the test set, the optimized model correctly labelled (without fault) 13/15 of the reports, up from 11/15 using the baseline model. That represented a 13.3% uplift in performance. Below is an example of what that looked like:

<img width="2337" height="3445" alt="image" src="https://github.com/user-attachments/assets/ea31e35f-9e63-4692-99de-af0e4f7d824d" />

Gold represents the human-labelled version (ground-truth). We see that with the improved prompt that the model was able to better detect that Company-level statements can be ignored when Group-level statements exist. This is a nuanced difference and it's pretty impressive that the model was able to learn that. 

Below is a snapshot of the overall improvement of the model on 10 of the items in the test set:

```
               pdf_name  score_baseline  score_optimized  improvement
EverythingFresh_2021...        0.611111         1.000000     0.388889
Access-Financial-Ser...        0.888889         1.000000     0.111111
Blue-Power-Group-Lim...        0.666667         0.777778     0.111111
2023-Cargo-Handlers-...        1.000000         1.000000     0.000000
Eppley-Quarterly-Rel...        1.000000         1.000000     0.000000
IronRock-Annual-Repo...        1.000000         1.000000     0.000000
Future-Energy-Source...        1.000000         1.000000     0.000000
2023-June-30-DCOVE-Q...        1.000000         1.000000     0.000000
JMMB-Group-Limited-U...        0.888889         0.888889     0.000000
Annual-Report-2022-D...        1.000000         1.000000     0.000000

```
