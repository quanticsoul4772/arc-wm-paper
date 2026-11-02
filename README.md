# What replay tests miss: verifying a clean-room rewrite of an LLM agent harness

ARC Prize 2026 (ARC-AGI-3), paper track.

- `writeup.md`: the Kaggle writeup (about 1,450 words); `cover.png`: its cover image.
- `paper.pdf` and `paper.md`: full method and results. Covers the deterministic equivalence checks, the pre-registered live tests, and nine controlled tests of harness mechanisms on the rewrite, including MTP speculative decoding.
- Code: https://github.com/quanticsoul4772/arc-wm-agent (CC0 1.0).

## Attribution

The rewrite re-implements the behavior of The Duck by Tufa Labs ([`Tufalabs/duck-harness`](https://github.com/Tufalabs/duck-harness), winning solution to ARC-AGI-3 Milestone 1). It was written from a behavioral spec, without access to that source. As of 2026-11-01 that repository has no LICENSE file; its `ARC3-Inference` package declares an MIT license classifier. Model: Qwen/Qwen3.6-27B (Apache-2.0).

## License

The text of this repository is released under CC0 1.0 (see `LICENSE`).
