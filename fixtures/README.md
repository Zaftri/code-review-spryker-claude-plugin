# Regression fixture corpus

Each directory is one past MR that produced a rule change, and exists so a future rule
edit can be checked for **both** false negatives and false positives before it lands
(`/spryker-review:learn` Step 5.6).

```
fixtures/<TICKET>/
  diff.patch    # the MR diff, from: glab api projects/<enc>/merge_requests/<iid>/changes
  comments.md   # the human reviewer notes, verbatim, with file:line and resolved state
  expected.md   # MUST fire / MUST NOT fire tables, plus any fix-destination constraints
```

`expected.md` is the important file. A corpus that only records what *should* fire
catches false negatives and misses the more common failure — a broadened rule that starts
firing on unrelated code. Always fill in the MUST NOT table, and always include at least
one site that is superficially similar to the trigger but legitimately correct.

Record the `rule_sheet_revision` the baseline ran under, so a later reader can tell
whether a miss was a genuine gap or simply predated the rule.
