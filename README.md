# build-metrics (composite) Action

> Send Github Actions build metrics to CloudWatch logs

*THIS IS CURRENTLY A COMPOSITE ACTION, NOT A STANDARD ONE*
## Usage
To use this composite action, add it as a last job in your workflow, using the following method:

### Pre-reqs
You will need to add a step to the other jobs in your workflow
```yml
# Add this step to the end of ALL previous jobs unless you have an always() job
# If not added to all jobs, the status will not be set on failure of a job
- name: Set status var for metrics
  if: always()
  # You might need to compute the status via some other method in some cases
  run: echo "WORKFLOW_STATUS=${{job.status}}" >> $GITHUB_ENV
```

Then you can add the metrics job at the end
```yml
send-build-metrics:
  # If you have multiple jobs that run in parallel (IE don't use needsarray ) add them all here
  # Without the needs value set properly, this job could run in parallel with another, sending
  # metrics before the workflow is actually complete
  # If `last-job-in-workflow` is run as the last one via it's own needs array, you can ref JUST it
  needs: [ first-job-in-workflow, second-job-in-workflow, last-job-in-workflow ]
  # Always send metrics
  if: always()
  runs-on: ubuntu-latest        
  # In the case that there is an error in the metrics reporting (bad tokens, API limits, etc)
  # we don't want to fail the whole workflow. continue-on-error not supported in the build-metrics
  # action itself (https://github.com/actions/runner/issues/1457) - so need to add it here
  continue-on-error: true
  steps:
    - name: Get build metrics repo
      uses: actions/checkout@v2
      with:
        # Don't need the full git history, just the latest commit
        fetch-depth: 0
        repository: slaughtr/cloudwatch-build-metrics
        # Clone the repo to the specified dir
        path: .github/actions/build-metrics
    - name: Run build-metrics action
      if: always()
      # Have to point to where the action.yml is
      uses: ./.github/actions/build-metrics/
      with:
        # This GITHUB_TOKEN value works for the action, no need for PAT
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        # If you didn't manage to set WORKFLOW_STATUS using the above step, it will report `undefined`
        WORKFLOW_STATUS: ${{ env.WORKFLOW_STATUS }}
        LOG_GROUP_NAME: ${{ env.LOG_GROUP_NAME }}
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        AWS_REGION: 'us-west-2' # this defaults to us-west-2, which is where we want things, no need to send at the moment
```

### Gotchas
If you send invalid credentials, the action will fail. Credentials need to be valid and scoped correctly. This applies to both Github tokens and AWS keys.

With `continue-on-error` it should not fail your workflow if the metrics fail to send. Without it, your workflow will report failed if the metrics fail. Switch it according to what fits your needs.

The metrics sending can fail for a number of reasons: invalid credentials, AWS API limits (`create-log-stream` in particular), invalid JSON from the github API, running in the wrong environment (`macos` or `windows` will likely not work without modifications due to unix tools used), and possibly if a repo is named in some particularly odd fashion.

In the case of the action failure, you should see error messages in the actions summary UI, as well as descriptive errors in the build logs.

## Development
Edit `action.yml`. The `.github/workflows/build-metrics*.yml` actions will run when you push your changes and do some basic testing for you.

# Acknowledgements
I created this while working at Tavour. Thank you @tavour for letting me open source this!