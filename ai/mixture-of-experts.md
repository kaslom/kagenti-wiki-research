---
tags: [paper, architecture, scaling]
---
# Mixture of Experts

Source: [Switch Transformers](https://arxiv.org/abs/2101.03961) (Fedus et al., 2021)

## Summary

Mixture of Experts routes each token to a subset of specialist sub-networks,
enabling model scaling without proportional compute increase.

## Key Ideas

- Sparse gating: each token activates only top-k experts
- Load balancing loss prevents routing collapse
- Capacity factor limits tokens per expert

## Results

- Switch Transformer: 7x speedup over T5-Base at same compute
- Mixtral 8x7B: competitive with GPT-3.5 using 2 active experts per token