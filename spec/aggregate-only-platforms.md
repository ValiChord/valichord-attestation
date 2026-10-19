# Guidance: platforms that store only an aggregate result

**Informative, not normative.** Added 2026-10-19 in answer to a question raised by Hawthorn on
`future-agi/future-agi#1368` (2026-08-17). It applies to any evaluation platform that keeps one row per
run with the headline score, while per-item results live somewhere else or nowhere.

## The short answer

The Merkle tree is built from **per-item results**, never from the aggregate. The aggregate is what
the tree exists to back up. A run that has no per-item results cannot produce an honest bundle.

## If per-item results exist somewhere else (e.g. another table or store)

1. **Leaves come from the per-item store.** One leaf per item that contributed to the aggregate.
2. **Put in each leaf everything the metric is computed from.** That means a stable item
   identifier, the per-item value, and anything that changes it (for an LLM-judged item, the
   judge's verdict and reason). A verifier recomputes the aggregate from disclosed leaves
   (`valichord_attestation.metrics`); a leaf missing the value makes that impossible. Prefer an item
   identifier that is bound to the input itself, such as a hash of the dataset row, so a leaf
   cannot be reattached to a different question.
3. **Fix the order.** The root depends on leaf order. Define it in your export, for example
   "sorted by item identifier", and never rely on database return order.
4. **Count honestly.** `samples.completed` is the number of leaves. `samples.total` is the number
   of items the run set out to evaluate. Items that errored or were skipped make
   `total > completed`. Do not drop them silently (conformance §4.1(10)).
5. **Freeze at completion.** Compute the root when the run completes, from the per-item results
   as they are at that moment. If that store can change or expire (retention windows, updates,
   re-scoring), the root becomes uncheckable once it does, possibly years before anyone notices.
   So keep **both**:
   - on the aggregate row: the root, `samples.completed`, and an identifier of the leaf source
     (which store, which snapshot);
   - a detached artifact holding the exact leaf list, written at completion and kept as long as
     the result is cited.

   This also answers "persist on the row, or a detached artifact?": the row anchors the claim,
   and the artifact is what makes the anchor verifiable.

## If there are no per-item results at all

There is no honest bundle to make.

- **Do not build a one-leaf tree out of the aggregate.** It would be well-formed, and it would
  commit to nothing a challenge could check, which is the reason the format refuses an empty
  sample list (conformance §4.1(8)). A single synthetic leaf is the same thing in disguise.
- Publish the number as what it is: an aggregate with no attestation behind it.
- The fix is upstream. Keep per-item results at evaluation time, even only as the minimal
  leaf described above.

## How a verifier then checks it

1. Challenge the holder (`Challenge`, `build_response`, `verify_response`). This proves the
   challenged leaves were committed under the root.
2. The holder discloses those leaves' contents. `bind_disclosed_samples` checks each one against
   its committed leaf.
3. `check_mean_metric` asks whether the aggregate is plausible given those leaves. With the whole
   leaf list, `check_metric` checks it exactly.
