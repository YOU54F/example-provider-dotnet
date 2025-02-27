
# Azure Pipelines - Setting up `contract_requiring_verification_published` webhook

1. Create Azure PAT token. [Docs](https://learn.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate?view=azure-devops&tabs=Windows)
2. Base64 encode, as Basic auth. 
    - <username:password>
    - `username` is blank
    - `password` is your PAT token
3. Setup New Azure Pipeline

```yaml
trigger: none

parameters:
- name: PACT_URL
  type: string
- name: PACT_WEBHOOK_MESSAGE
  type: string
  default: "Run tests"

pool:
  vmImage: ubuntu-latest

steps:

- script: make ci_webhook
  displayName: '✅ ${{parameters.PACT_WEBHOOK_MESSAGE}}'
  env:
    PACT_URL: '${{parameters.PACT_URL}}'
    PACT_BROKER_BASE_URL: https://testdemo.pactflow.io
    PACT_BROKER_TOKEN: $(PACT_BROKER_TOKEN)
    PACT_BROKER_PUBLISH_VERIFICATION_RESULTS: true  
    GIT_COMMIT: $(Build.SourceVersion)
    GIT_BRANCH: $(Build.SourceBranchName)
```
3. Once created, edit the existing workflow
  - `Edit` -> `Variables`
    - Add `PACT_BROKER_TOKEN` as `secret` type
    - Select `...` -> `Triggers` -> `Variables` -> `system.definitionId` 
      - This is your `<pipelineId>`

4. Setup Pact Broker webhook using the following template.
  - A json payload is provided in `azure-pipelines-contract_requiring_verification_published-webhook.json`
  - Url template `https://<instance>/<organization>/<project>/_apis/pipelines/<pipelineId>/runs?api-version=6.0-preview.1`
  - `<instance>` usually `dev.azure.com`
  - `<organization>` azure org name
  - `<project>` azure project name
  - `<pipelineId>` azure pipeline id, for newly created pipeline. see `system.definitionId` noted earlier

```json
{
  "request": {
    "method": "POST",
    "url": "https://<instance>/<organization>/<project>/_apis/pipelines/<pipelineId>/runs?api-version=6.0-preview.1",
    "headers": {
      "Content-Type": "application/json",
      "Accept": "application/json",
      "Authorization": "Basic TOKEN"
    },
    "body": {
      "resources": {
        "repositories": {
          "self": {
            "refName": "${pactbroker.providerVersionBranch}",
            "version": "${pactbroker.providerVersionNumber}"
          }
        }
      },
        "templateParameters": {
            "PACT_URL": "${pactbroker.pactUrl}",
            "PACT_WEBHOOK_MESSAGE": "Verify changed pact for ${pactbroker.consumerName} version ${pactbroker.consumerVersionNumber} branch ${pactbroker.consumerVersionBranch} by ${pactbroker.providerVersionNumber} (${pactbroker.providerVersionDescriptions})"
        }
    }
  }
}
```