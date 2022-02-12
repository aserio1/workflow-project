# workflow-communications-poc

Simple POC to display ways to share information beetween github actions workflows

The challenge here is to share information between different github actions workflows without changing the workflow itself.

So to do this, we will extract the logs from the target workflow and use a series of REGEX patterns to extract the information we need.

In this case, we will extract the following information:

    - The commit hash printed in workflow-1
    - The deploy id printed in workflow-1
    - The status of the deployment printed in workflow-1

## workflow-1

This first workflow will print the commit hash and the deploy id of the deployment. That info will be extracted in workflow-2.

```yaml
name: workflow-1
on:
  workflow_dispatch:
    branches:
      - main

jobs:
  Job1:
    name: Generating strings on the output
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Generate output strings
        run: |
          echo "commithash id is ${GITHUB_SHA}"
          echo "499411494"
          echo "pending"
```

## workflow-2

In this workflow, we will extract the commit hash and the deploy id from the workflow-1 output.

To do that we will use a series of REGEX patterns to extract the information we need.

Check the comments in the code to see how the patterns are defined.

```yaml
name: workflow-2
on:
  workflow_dispatch:
    branches:
      - main

jobs:
  Job2:
    name: Reading output from workflow 1
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Get output from workflow 1 using gihub cli
        run: |

          # Use github CLI (gh) to get the ID of last completed execution of workflow-1 
          RUN_ID=$(gh workflow view workflow-1.yml | grep -m1 "completed" | awk '{print $8}')
          # With the run id from the last execution of workflow-1, get the output from the workflow-1 and save it to a file output.txt      
          gh run view ${RUN_ID} --log > output.txt
          #sanitize the output file and save the contents in a new file sanitized_log.txt
          sed 's/^.*Z //g' output.txt > sanitized_log.txt
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Read output from workflow 1
        run: |

          #From the sanitized log file, extract the commit hash 

          COMMIT_HASH=$(grep '^commithash id is ' sanitized_log.txt | sed 's/commithash id is //g')
          echo "Commit hash from workflow-1 is ${COMMIT_HASH}"

          #From the sanitized log file, extract the deploy id

          DEPLOY_ID=$(grep -E '^[[:digit:]]{9}' sanitized_log.txt )  
          echo "Deployment id from workflow-1 is ${DEPLOY_ID}"

          #From the sanitized log file, extract the status of the deployment

          STATUS=$(grep -E '^(pending|queued)' sanitized_log.txt)
          echo "Status from workflow-1 is ${STATUS}"
```
