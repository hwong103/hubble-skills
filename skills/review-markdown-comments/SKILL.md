---
name: review-markdown-comments
description: "Address Hubble review comments stored inline in Markdown CriticMarkup: locate a comment by id or exact anchor, append agent replies, preserve unknown metadata and unrelated Markdown, and resolve only after addressing the request. Use when an agent is asked to reply to, resolve, or reopen a Hubble review comment in a Markdown file."
---

# Review Markdown Comments

Use this skill when a Hubble user asks an agent to address a review comment in a
Markdown file. Work directly on the file named by the user; do not require the
Hubble UI, browser automation, Rough Draft, or a sidecar database.

## Comment format

Hubble stores review state in the Markdown body using this portable subset:

```md
{==anchored text==}{>>comment body<<}{#c1}
{++suggested insertion++}
{--suggested deletion--}
{~~original text~>replacement text~~}
```

Thread state may follow a comment anchor as an encoded HTML comment:

```md
<!-- hubble-review:%7B%22replies%22%3A%5B%5D%2C%22resolved%22%3Afalse%7D-->
```

The decoded metadata is a JSON object. Hubble uses `replies` and `resolved`,
but other top-level keys are allowed and must survive edits. A reply has this
shape:

```json
{
  "id": "r1",
  "body": "Agent response",
  "author": "agent",
  "createdAt": "2026-01-01T00:00:00.000Z"
}
```

## Address a comment

1. Read the complete file before editing it.
2. Locate the requested compact id, such as `{#c7}`. If the user gives only
   anchored text, locate the exact CriticMarkup anchor instead. Confirm the
   anchored text and comment body match the user’s request. If the id or exact
   anchor is absent or ambiguous, stop and report the ambiguity.
3. Inspect the metadata immediately following that logical comment. If it is
   absent, start with an empty metadata object and add a new encoded object
   after the `{#cN}` marker. If it is present, decode it with URI decoding and
   JSON parsing; if it is malformed, stop rather than overwrite it. Do not
   hand-edit an encoded substring if a structured edit is practical.
4. Preserve the existing metadata object and every existing reply object.
5. Append a new reply with the next unused `rN` id, `author: "agent"`, the
   response body, and the current UTC time as an ISO-8601 `createdAt` value.
6. Re-encode the complete metadata object with URI encoding and write it back
   immediately after the comment’s `{#cN}` marker. For a fragmented anchor,
   place the single authoritative metadata block after the final fragment.
7. Set `resolved: true` only when the user’s request has actually been
   addressed. For acknowledgements, tests, questions, or partial work, keep
   `resolved: false`.

## Preservation rules

- Keep the anchored text byte-for-byte unchanged.
- Keep the comment body, compact anchor id, CriticMarkup delimiters, suggested
  edits, links, images, front matter, and unrelated Markdown unchanged.
- Never remove or rewrite unknown metadata keys.
- Never replace existing replies; append only.
- If one logical comment id appears in multiple CriticMarkup fragments, treat
  the final fragment as authoritative. Preserve every marker and every metadata
  field, but move the single existing metadata block after the final fragment
  when it currently follows an earlier fragment; never create duplicate blocks.
  Report the fragmented anchor if it may affect the user’s formatting.
- Review markers inside inline code or fenced code are literal text and must
  not be treated as comments.

## Validation

After editing:

1. Re-read the file.
2. Confirm the requested id and exact anchored text are still present.
3. Decode the metadata after the final fragment and verify the new reply is the
   final reply, the old replies and unknown keys remain, and `resolved` has the
   intended value.
4. Check the surrounding Markdown for accidental changes. For a tracked file,
   inspect `git diff -- <file>`; for an ignored/generated file, compare the
   before and after text or inspect the exact changed line.
5. Report the reply id and whether the comment remains unresolved.

Use a concise acknowledgement when the comment is only a test, for example:
“Received the test comment; the anchored text and surrounding Markdown are
unchanged.”
