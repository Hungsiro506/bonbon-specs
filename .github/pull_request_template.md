## Summary

<!-- 1–3 sentences: what changes and why. -->

## Type of change

- [ ] New feature spec (new `<nnn>-<slug>/` folder)
- [ ] New acceptance scenario(s) appended to an existing spec
- [ ] Prose edit on an existing AC (number unchanged)
- [ ] New `FR-xxx` appended
- [ ] Deprecation (marked `**DEPRECATED**`, not deleted)
- [ ] Meta / README / tooling

## Append-only checklist

- [ ] I did **not** renumber any existing acceptance scenario.
- [ ] I did **not** delete any existing acceptance scenario.
- [ ] I did **not** insert new items in the middle of a numbered list.
- [ ] New ACs / FRs are appended at the end of their respective lists.

If the answer to any of the first three is "no", describe why — renumbering has downstream impact on bonbon tests.

## Downstream impact

- [ ] I have flagged engineering on changes that may affect existing tests (prose-only edits and deprecations).
- [ ] No bonbon test currently references an AC I'm removing (or a port PR is queued).
