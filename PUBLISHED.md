# This repository is published, not authored

`.github/workflows/preview.yml` and `README.md` are copied from
`github-actions/preview-workflow/` in the churner monorepo, which is where
they are edited, reviewed and tested. Do not patch them here: the next
release overwrites the file, and the change would never have run against the
workflow's test suite (`shared/tests/preview-workflow.test.ts`).

From v6 previews run on Fargate only, through the preview stack's controller
function; the workflow fetches no script from anywhere.

Released from churner monorepo commit `9e4f625a1`.
