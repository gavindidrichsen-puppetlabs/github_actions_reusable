# workflow-restarter

## Description

## Usage

To trigger this action from within a workflow, call in a way similar to the `steps` snippet below.

```yaml
jobs:
  acceptance:
    steps:
      ...
      ...
  
  on-failure-workflow-restarter:
    needs: [acceptance]                                     # (1) "acceptance" job must run before this one
    if: always() && needs.acceptance.result == 'failure'    # (2) ONLY IF the "acceptance" job fails, then run this one
    runs-on: ubuntu-latest
    steps:
      # (3) checkout THIS repository since I'm "using" a local custom action
      - name: Checkout repository
        uses: actions/checkout@v2

      # (4) now "use" the custom action to retrigger the failed "acceptance job" above
      - name: Trigger reusable workflow
        uses: ./.github/actions/workflow-restarter-proxy
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          repository: ${{ github.repository }}
          run_id: ${{ github.run_id }}
```

Notice:

* There are a couple pre-quisites before using the custom action.  **(1)** you must specify what preceeding job this `on-failure-cleanup` "needs"; and **(2)** you must ensure that `on-failure-cleanup` only runs on failure of the preceeding job.  In other words, the following ensures that `on-failure-cleanup` is one, triggered after the `acceptance` job and two, only triggered if `acceptance` fails.

```yaml
    needs: [acceptance]
    if: always() && needs.acceptance.result == 'failure'
```

Further:

* The `actions/checkout@v2` must proceed the call to the custom action.  Without this, the current workflow can't "see" the custom action.  Since this custom action resides within this repository then we simply checkout the same repo.  If, however, the custom action lives somewhere else then another reference mechanism would be required.
* The `uses: ./.github/actions/workflow-restarter-proxy` points to the custom action in this repository.
* The `GITHUB_TOKEN` is passed in as an environment variable for a couple of reasons.  One, this is one way to safely pass a secret:  since custom actions don't provide a "secrets" hash, then using an environment variable is the next best way to safely pass this token.  Two, this token is required by the retry-workflow:  Since the retry-workflow will in most cases be in a separate repository, then it will need the `GITHUB_TOKEN` from the source repository/workflow to make re-try the source workflow.
* The `repository` and `run_id` are required by the custom action.  Without these it cannot reference the workflow that needs to be restarted.
