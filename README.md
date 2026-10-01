# merge-queue-ci-skipper

GitHub Merge Queues are very useful to avoid the need for pull request authors to keep pressing "update branch" in their PR
until they can merge their own PR. This happens in large, contested repositories, especially monorepos, where PRs are merged
very frequently.

However, Merge Queues have one disadvantage for repositories with long-running CI checks: When a PR is enqueued into the
Merge Queue, all CI checks are run again (in the `merge_group` context). This is because other PRs could have been ahead of them
in the queue, in which case the Merge Queue automatically rebases the changes of the PR onto the changes of the higher-ranked PR.
Theoretically, the changes from both PRs combined could cause tests to fail, so it is safer to run the CI checks again.

However, there is one scenario in which running the checks again is redundant. When a PR is enqueued into an empty merge queue,
it immediately is the head of the queue. If the PR was already up-to-date with the target branch, the CI checks are run on
identical repository content. This causes unnecessary waiting and compute times.

The `merge-queue-ci-skipper` GitHub Action lets you avoid this issue.

## Integration

### From this repository

First, add the following step in the beginning of your job:

```yml
- id: merge-queue-ci-skipper
  uses: cariad-tech/merge-queue-ci-skipper@main
  with:
      secret: ${{ secrets.GH_ACCESS_TOKEN }}
```

To get a stable build, please replace the version (`main`) with the latest released version.

### From your own repository

Alternatively, you may copy the `action.yml` into the following path in your repository: `.github/merge-queue-ci-skipper/action.yml`

After that, configure the step like so:

```yml
- id: merge-queue-ci-skipper
  uses: uses: ./.github/merge-queue-ci-skipper
  with:
      secret: ${{ secrets.GH_ACCESS_TOKEN }}
```


### Configuration

The checks required by the target branch are looked up, and every one of them has to be present and successful on the
head of the pull request before its merge queue checks are skipped.

A branch can require checks through either of two separate features, and both are consulted:

- **Rulesets.** Read with the workflow `GITHUB_TOKEN`, needing no configuration and no extra permissions. Repository,
  organisation and enterprise rulesets are all covered, because the endpoint used reports the rules that apply to the
  branch with their conditions already evaluated.
- **Classic branch protection.** Reading this needs `administration:read` permission, which cannot be granted to
  `GITHUB_TOKEN`, so it is only consulted when a PAT or app token is passed as the optional `secret` input.
  `GH_ACCESS_TOKEN` is such a token, and has to be issued by a user with admin permissions for the repository. Omit the
  input if the branch is protected only by a ruleset.

If neither of them requires a check (which is also what an unset `secret` looks like for a branch protected classically),
a warning is logged and every check run on the head of the pull request has to have passed or been skipped instead.

This GitHub Action requires `read` permissions for `pull-requests`, `contents` and `checks`.

Next, add the following conditional to _every_ workflow step that should be _skipped_ if the conditions outlined in the scenario
described above are true:

```yml
- name: Some build step
  if: ${{ steps.merge-queue-ci-skipper.outputs.skip-check != 'true' }}
  run: ./gradlew assemble
```

## Legal Disclaimer

_This Software / Contribution is unfinished, untested and is in particular not in a state to be
used in any productive or series context. It is provided as a starting point for further
development and any use requires extensive checks, improvements and testing, for which
exclusively the user integrating this OSS shall be responsible._
