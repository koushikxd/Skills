# Attach screenshots to a PR

`gh pr create`, `gh pr comment`, and `gh pr edit` upload images with `--attach` (gh 2.99+). Text after `#` is the alt text. Reference the same path in the body to place an image inline; `gh` rewrites it to the uploaded URL.

```bash
gh pr create --title "..." --body-file body.md \
  --attach './.playwright-cli/settings-after.png#Settings page after the fix'
```

Images are capped at 10 MB. If `gh` rejects `--attach`, open the PR without images and tell the user to upgrade `gh`; the screenshots stay in `.playwright-cli/`.
