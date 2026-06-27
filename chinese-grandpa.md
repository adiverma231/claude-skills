---
name: chinese-grandpa
description: Compress a long instruction brief into dense Mandarin before it goes out to a fleet of sub-agents, so big parallel/repeated runs cost fewer tokens without losing intent. Use whenever the user runs /chinese-grandpa, or says they're "sending the same instructions to a bunch of sub-agents," wants to "compress/shrink a prompt," "cut the token bill," "make agent runs cheaper or faster," or is fanning one long brief out across many agents. Translate-only: it returns a compact Mandarin brief to reuse and does NOT dispatch the agents itself. Preserves code, file paths, commands, and proper nouns verbatim, and verifies intent survived before handing it back.
---

# Chinese Grandpa

Turn a long English brief into the Mandarin your grandfather would use: the same orders in a fraction of the characters, and never vague about what he means. You hand back **one compact Mandarin brief** the user pastes into each sub-agent. Mandarin packs more meaning per token, so when the same brief is fanned out across many agents, the savings multiply while the translation is paid for once.

This skill is **translate-only**. Produce the compressed brief; do not spawn or dispatch the agents.

## When this is worth it (and when it isn't)

Worth it: a long, prose-heavy brief that will be reused across many sub-agents or many runs. The one-time translation cost amortizes across every agent that reads it, so savings scale with fleet size.

Skip it (say so plainly): a short brief, a single agent, or a brief that is mostly code, paths, and literals — there is little prose to compress, so the gain won't cover the round-trip effort. If measured savings come back under ~15%, tell the user it isn't paying for itself here.

## Preserve verbatim — do not translate

Translating an execution token silently breaks every agent downstream, and the failure is invisible until the run fails. Splice these back in **byte-for-byte**, exactly as written:

- Code (blocks and inline), regexes, and exact string literals the agent must match or emit
- File paths, directory names, CLI commands and flags, env-var names
- API / function / class / variable / library names, URLs, emails
- Numbers, IDs, hashes, version strings, dates, units
- Structural markers the downstream expects (JSON keys, XML/HTML tags, markdown the agent must output)
- Proper nouns where a translated/transliterated form would be ambiguous (product, brand, repo names)

Everything else — prose instructions, rationale, role descriptions, constraints, tone and style guidance, repeated boilerplate — is fair game to compress.

## How to compress (adaptive register)

Default to concise modern written Mandarin (简体, 书面语). Drop the connective filler Chinese omits naturally — pronouns, articles, "please," hedges — these carry no instruction. Lean terser on low-risk boilerplate; stay fully explicit on anything precise, conditional, or negated ("if X then Y, else Z"), because a dropped 不 or a collapsed condition flips the instruction. Do not drift into classical 文言文: density that buys ambiguity costs more than it saves.

The grandpa test for each line: *most meaning, fewest characters, zero ambiguity.* If terseness adds doubt, add the character back.

Two things the compressed brief must always carry, because the agent sees only this brief and never the English:
- It must stand alone — everything an agent needs to do the job, nothing assumed from the original.
- Keep any required **output language/format** as an explicit line (e.g. 输出用英文 / output in English). Instructing in Mandarin does not require the agent to answer in Mandarin, but make the target explicit so it can't drift.

## Procedure

1. **Segment.** Split the brief into preserve-verbatim spans (see list above) and translatable prose.
2. **Compress.** Render the prose as compact Mandarin; splice the preserved spans back in unchanged.
3. **Round-trip check (do this before showing anything).** Back-translate your Mandarin to English and compare against the original. Confirm: every actionable instruction survived; no constraint, negation, or conditional flipped; all numbers and preserved tokens are byte-identical; output-language/format requirement intact. Fix and recheck until clean. This is cheap insurance against a lossy instruction multiplying across the whole fleet.
4. **Measure.** Write the original to `orig.txt` and the compressed brief to `zh.txt`, then run the snippet below to get real numbers for this brief.
5. **Deliver** in the output format below.

## Measure savings

```bash
pip install tiktoken --break-system-packages -q 2>/dev/null
python3 - "$@" <<'PY' orig.txt zh.txt
import sys
def toks(p):
    s=open(p,encoding="utf-8").read()
    try:
        import tiktoken; return len(tiktoken.get_encoding("o200k_base").encode(s)), len(s)
    except Exception:
        return None, len(s)
te,ce=toks(sys.argv[1]); tz,cz=toks(sys.argv[2])
if te and tz: print(f"tokens: {te} -> {tz}  (saved {te-tz}, {(te-tz)/te*100:.0f}%)")
else:         print("tokens: tiktoken unavailable — see chars")
print(f"chars:  {ce} -> {cz}  (saved {ce-cz}, {(ce-cz)/ce*100:.0f}%)")
PY
```

`o200k_base` is a stand-in for Claude's tokenizer — close enough to trust the **trend and the percentage**, not the exact count. For an exact Claude figure the user can run Anthropic's `count_tokens`. Savings shown are per-agent; multiply by the number of sub-agents for the real bill impact.

## Output format

Keep your own commentary to one line. The user wants the artifact, not a writeup. Return exactly:

1. The compressed brief in a fenced block, ready to copy:
```
<Mandarin brief, preserved tokens inline>
```
2. Savings: `tokens A→B (−N%, ×M agents) · chars A→B (−N%)`
3. `Round-trip: intact` — or one line naming what you adjusted to keep intent.

## Edge cases

- **Mostly-code brief / savings < ~15%:** report it and recommend skipping — the compression isn't earning its keep.
- **Mixed-language input:** leave existing non-English content as-is; only compress the English instructions.
- **Tables / structured data:** preserve structure and data values; compress only label/instruction prose, never the data.
- **Traditional Chinese needed downstream:** switch to 繁體 on request; default is Simplified.
- **Tiny brief:** the round-trip + translation overhead can exceed the savings; say so rather than forcing it.
