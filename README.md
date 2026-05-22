# auto-tag

Shared GitHub Actions reusable workflow that date-stamps every push with a
`YYMMDD.PUBLIC.BUILD` tag and optionally cuts a public GitHub Release.

Used across the [parasxos](https://github.com/parasxos) workspace. Public so it
works for callers of any visibility (the previous home was a private repo,
which GitHub disallows public callers from reaching).

## Use it

In a consuming repo, add `.github/workflows/auto-tag.yml`:

```yaml
name: Auto Tag on Push

on:
  push:
    branches: [main, master]
  workflow_dispatch:
    inputs:
      public_release:
        description: 'Create a public release (increment PUBLIC version)'
        required: true
        default: 'true'
        type: boolean

permissions:
  contents: write

jobs:
  determine-release-type:
    name: Check Release Type
    runs-on: ubuntu-latest
    if: "!contains(github.event.head_commit.message, '[skip tag]')"
    outputs:
      is_public: ${{ steps.check.outputs.is_public }}
    steps:
      - uses: actions/checkout@v5
        with:
          fetch-depth: 1
      - id: check
        run: |
          IS_PUBLIC="false"
          if [ "${{ github.event_name }}" = "workflow_dispatch" ] && [ "${{ inputs.public_release }}" = "true" ]; then
            IS_PUBLIC="true"
          fi
          if [ "${{ github.event_name }}" = "push" ]; then
            COMMIT_MSG=$(git log -1 --pretty=%B)
            if echo "$COMMIT_MSG" | grep -qi '\[release\]'; then
              IS_PUBLIC="true"
            fi
          fi
          echo "is_public=$IS_PUBLIC" >> $GITHUB_OUTPUT

  auto-tag:
    name: Create Tag
    needs: determine-release-type
    uses: parasxos/auto-tag/.github/workflows/auto-tag-reusable.yml@main
    with:
      is_public_release: ${{ needs.determine-release-type.outputs.is_public == 'true' }}
      runs_on: ubuntu-latest
```

## Commit message markers

- `[release]` — cut a public release (increments PUBLIC, resets BUILD to 1)
- `[skip tag]` — suppress tagging entirely

Default push increments BUILD only.

## Tag format

`YYMMDD.PUBLIC.BUILD` — e.g. `260522.0.1`, `260522.1.1` (first public release of the day).

## Pinning

`@main` follows the latest. For reproducibility, pin to a commit SHA instead:

```yaml
uses: parasxos/auto-tag/.github/workflows/auto-tag-reusable.yml@<sha>
```

## License

MIT.
