# Power Apps Deploy Solution

This Action deploys a solution file in the Release with matching tag into the target environment.

## Inputs

| Input | Required | Default | Description |
| ----- | -------- | ------- | ----------- |
| token | ✅ | - | [GitHub Personal Access Token](https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token). This is required for downloading the solution files from Releases. |
| tag | ✅ | - | Git Tag associated with the Release. If you are using [Power Apps Solution Extract Action](https://github.com/rajyraman/powerapps-solution-extract), it tags the Release with the Solution's version number. |
| solutionName | ✅ | - | The Unique Name of the solution. This is used in Solution Upgrade step, if you are deploying a Managed Solution. |
| targetEnvironmentUrl | ✅ | - | Environment URL where the Solution will be deployed to. |
| applicationId | ✅ | - | Application Id that will be used to connect to the Power Apps environment.Refer [Scott Durow's video](https://www.youtube.com/watch?v=Td7Bk3IXJ9s) if you do not know how to set Application Registration and scopes in Azure to facilitate this. |
| applicationSecret | ✅ | - | Secret that will be used in the Connection String for authenticating to Power Apps. Use GitHub Secrets on your repo to store this value. Refer to [Using Encrypted Secrets](https://docs.github.com/en/actions/configuring-and-managing-workflows/|creating-and-storing-encrypted-secrets#using-encrypted-secrets-in-a-workflow) on how to use this in your workflow. |
| managed | ✅ | false | By default, this Action will import Unmanaged solution. Set this parameter to true, if you want to import Managed Solution. |


## Outputs

| Output | Description |
| ----- | ----------- |
| solution | File Name of the Solution that was imported. |

## Example

Below is an example on how to use [Power Apps Solution Extract]() and [Power Apps Deploy Solution]() in your Workflow.
```yaml
# This workflow is run only manually, as it uses a workflow_dispatch. It is recommended to use cron or push into main branch to trigger this. Since this workflow also commits into the repo, use ignore tags to prevent infinite loop. Refer https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions#example-ignoring-branches-and-tags

name: extract-and-deploy
on:
  workflow_dispatch:
    inputs:
      solutionName:
        description: "Solution Name"
        required: true
        default: 'GitHubSolution'
      sourceEnvironmentUrl:
        description: "Source Environment URL"
        required: true
        default: 'https://xxxx.crm.dynamics.com'
      targetEnvironmentUrl:
        description: "Target Environment URL"
        required: true
        default: 'https://xxxx.crm.dynamics.com'        
      release:
        description: "Release?"
        required: true
        default: true       
      managed:
        description: "Managed?"
        required: true
        default: true        
      debug:
        description: "Debug?"
        required: true
        default: 'true'                      

jobs:
  build:
    # The type of runner that the job will run on. This needs to be Windows runner.
    runs-on: windows-latest

    steps:
      - uses: actions/checkout@v2
        
      - name: Dump GitHub context
        if: github.event.inputs.debug == 'true'
        env:
          GITHUB_CONTEXT: ${{ toJson(github) }}
        run: echo "$GITHUB_CONTEXT"
                                
      - name: Extract Solution
        id: extract-solution
        uses: rajyraman/powerapps-solution-extract@v1.1
        with: 
          token: ${{ secrets.GITHUB_TOKEN }}
          solutionName: ${{ github.event.inputs.solutionName }}
          sourceEnvironmentUrl: ${{ github.event.inputs.sourceEnvironmentUrl }}
          applicationId: ${{ secrets.APPLICATION_ID }}
          applicationSecret: ${{ secrets.APPLICATION_SECRET }}
          releaseSolution: ${{ github.event.inputs.release }}
          
      - name: Deploy Solution
        id: deploy-solution
        uses: rajyraman/powerapps-deploy-solution@v1.1
        with: 
          token: ${{ secrets.GITHUB_TOKEN }}
          solutionName: ${{ github.event.inputs.solutionName }}
          targetEnvironmentUrl: ${{ github.event.inputs.targetEnvironmentUrl }}
          applicationId: ${{ secrets.APPLICATION_ID }}
          applicationSecret: ${{ secrets.APPLICATION_SECRET }}
          tag: ${{ steps.extract-solution.outputs.solutionVersion }}
          managed: ${{ github.event.inputs.managed }}    

        # This step below can be removed, if you do not want to send a notification to Teams about this solution deployment.
      - name: Notify Teams
        uses: fjogeleit/http-request-action@v1.4.1
        with:
          # Url to Power Automate Flow to notify about Solution Deployment
          url: "${{ secrets.FLOW_HTTP_URL }}"
          # Request body to be sent the Flow with HTTP Trigger
          data: >-
            {
                "solutionFile": "${{ steps.deploy-solution.outputs.solution }}",
                "tag": "${{ steps.deploy-solution.outputs.solutionVersion }}",
                "environmentUrl": "${{ github.event.inputs.targetEnvironmentUrl }}",
                "repoUrl": "https://github.com/${{ github.repository }}",
                "runId": "${{ github.run_id }}",
                "sha": "${{ github.sha }}"
            }
```