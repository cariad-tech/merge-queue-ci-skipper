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

First, add the following step in the beginning of your job:

```yml
- id: merge-queue-ci-skipper
  uses: cariad-tech/merge-queue-ci-skipper@main
  with:
      secret: ${{ secrets.GH_ACCESS_TOKEN }}
```

To get a stable build, please replace the version (`main`) with the latest released version.

Note the `secret` input: `GH_ACCESS_TOKEN` is a token that has `administration:read` permissions.
It needs to be issued by a user that has admin permissions for the repository.
This token is used to find the list of required branch checks and confirm that they already passed.
This input is optional and may be omitted as long as the repository has _Require status checks to pass before merging_
enabled. If this setting is enabled, GitHub will not enqueue the PR into the Merge Queue until all checks have passed.

This GitHub Action seems to require `read` permissions for `pull-requests` and `contents`.

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
