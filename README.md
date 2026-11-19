stages:
  - stage: DependencyScan
    displayName: 'Dependency Scan (Trivy)'
    jobs:
      - job: Trivy
        displayName: 'Scan composer.lock'
        pool:
          name: '${{ parameters.poolName }}'
        steps:
          - checkout: deps
            clean: true
            fetchDepth: 1
            path: deps

          - task: AzureCLI@2
            displayName: 'Login to base Image ACR'
            inputs:
              azureSubscription: '${{ parameters.baseServiceConnection }}'
              scriptType: bash
              scriptLocation: inlineScript
              inlineScript: |
                BASE_ACR=$(echo ${{ parameters.baseAcrLoginServer }} | cut -d'.' -f1)
                az acr login --name "$BASE_ACR"
