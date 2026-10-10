# Flowly: the prompt

[`PROMPT.md`](PROMPT.md) is a build specification for Flowly, written for a capable coding agent: what the
software is, its architecture, every feature area with the rules that matter, the non-functional requirements and
a suggested build order. Given to an agent milestone by milestone, it should produce software equivalent to the
current Flowly.

Use it to rebuild Flowly in another stack, to onboard contributors, or to see how the product is meant to behave.

## The code

- [flowly-backend](https://github.com/flowly-org/flowly-backend): Django API and worker
- [flowly-frontend](https://github.com/flowly-org/flowly-frontend): Angular web app, Android and iOS (Capacitor)
- [flowly-desktop](https://github.com/flowly-org/flowly-desktop): Tauri tray timer
- [flowly-org.github.io](https://github.com/flowly-org/flowly-org.github.io): the site, flowly.net

## Licence

AGPL-3.0, like the code. See [LICENSE](LICENSE).
