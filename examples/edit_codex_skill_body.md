got tired of jumping between two terminals every time i wanted codex's image_gen tool, so i wrote a small skill for claude code that wraps the three things i actually use codex for.

the image bit is the part most people seem to miss — codex cli has a built-in `image_gen.imagegen` that uses your chatgpt subscription auth. no OPENAI_API_KEY, no separate billing. it only fires if your prompt doesn't tell codex to "use the openai api" or shell out to curl/python, so the skill encodes the right invocation. on windows there's also a copy bug in 0.128.0 (`CreateProcessAsUserW failed: 5`) where the png generates fine but never lands in your workspace — the skill grabs it from `~/.codex/generated_images/` itself.

the other two modes are nothing fancy:

- `/codex-cli review` regroups codex's findings into critical/high/med/low with file:line cites instead of dumping raw output, and flags conflicts with anything claude already verified
- `/codex-cli impl "..."` runs `codex exec -s workspace-write -C <repo>`. never escalates to danger-full-access without explicit per-run ok

one markdown file plus install scripts, MIT, ships with a reproducible test suite.

https://github.com/cskwork/claude-codex-skill

needs claude code + codex >= 0.128 + `codex login`. happy to take questions or feedback.
