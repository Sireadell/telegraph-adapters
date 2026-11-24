# Telegraph adapters

LoRA adapter packages for Telegraph Protocol Evaluator intents. Each file is a small `.safetensors` adapter (about 6.8 MB) that sits on top of the network's `modernbert-base-lora16-v1` foundation model and compares two answers to the same prompt, picking the better one.

This repository holds the adapters we submitted as an Evaluator. The files are public so anyone can download one, check its hash against the registry, and run it.

## What is here

| Item | Detail |
|---|---|
| `adapters/` | 47 adapter files across 15 intents |
| File name | `<INTENT>-<first 8 hex characters of the file hash>.safetensors` |
| Intents covered | CODE_REVIEW, CODE_PATCH_VERIFY, CONTRACT_OBLIGATION_AUDIT, CUSTOMER_TICKET_RESOLUTION, EMAIL_SECURITY, LLM_OUTPUT_EVALUATION, OPTIMAL_EXECUTION_ROUTE, PRODUCT_AUTHENTICITY, SANCTIONS_SCREENING_MATCH, SECURITY_REVIEW, SENTIMENT_ANALYSIS, THREAT_INTELLIGENCE, VENDOR_VERIFY, WASH_TRADING_DETECTION, WEATHER_FORECAST_VERIFY |

Several intents have more than one file. Later files are retrains; the registry decides which one is live.

## Check a file against the registry

The hash the registry stores is the keccak256 of the file's bytes. To confirm that a downloaded file is the one a registry entry points to:

```js
// node check.mjs adapters/CODE_REVIEW-660d6f29.safetensors
import { readFileSync } from 'fs';
import { keccak256 } from 'ethers';
console.log(keccak256(readFileSync(process.argv[2])));
```

The first 8 hex characters after `0x` must match the number in the file name. Compare the full value with the `wasm_hash` of the entry at `https://devnode.telegraphprotocol.com/api/wasm`.

## How the adapters were trained and tested

Every adapter is trained on preference pairs: one prompt, one better answer, one worse answer. The two answers differ in one detail, so the adapter has to notice the detail instead of the style.

For CODE_REVIEW the pairs were built from public lint-rule documentation for Rust, Python and JavaScript, plus bug pairs made by changing one operator in a short function. The worse answer is one of:

- a review of the wrong finding for that code
- the right finding but the wrong category
- an approval of code that has a problem
- the right line number but a fix in the wrong direction
- the wrong line

The same prompts are written in several shapes (JSON, a list of findings, plain text, markdown, YAML-like text) and with the code laid out normally or squashed onto one line, so the adapter cannot lean on formatting.

Held-out results for the current CODE_REVIEW file (`CODE_REVIEW-660d6f29`), on pairs built only from material the adapter never trained on:

| Test set | Pairs | Correct |
|---|---|---|
| Hard set (one-detail differences, several shapes) | 128 | 118 (92%) |
| Out-of-distribution set (Rust bugs, answer shapes and prompt wording not seen in training) | 80 | 76 (95%) |

Weakest case: choosing the right lint category when the finding itself is right (3 of 10).

## Limits and lessons

- These are our own tests, built from public sources. They are not the network's hidden pairs, and a pass here does not guarantee a pass there.
- An earlier CODE_REVIEW build scored 60.71% on the network's 28 hidden pairs, under the 70% floor, even though it did well on our first test set. That test set was too easy: the two answers differed in obvious ways. Making the pairs differ by one detail, and testing on shapes the adapter had not seen, is what moved the numbers above.
- The adapter sees only the prompt and two answers, with no tools and no way to run the code, so short one-sided prompts are the hardest.

## Try it

1. Download any file from `adapters/`.
2. Check the hash as above.
3. Load it with the network's foundation model and engine (see the Telegraph docs), score a few pairs of your own, and see where it breaks. If it breaks somewhere interesting, open an issue with the pair.
