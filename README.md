parameters:
  - name: poolName
    type: string
  - name: baseServiceConnection
    type: string
  - name: baseAcrLoginServer
    type: string
  - name: trivyImage
    type: string
  - name: severity
    type: string
    default: 'CRITICAL,HIGH'
  - name: exitCode
    type: string
    default: '0'

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
                set -e
                BASE_ACR=$(echo ${{ parameters.baseAcrLoginServer }} | cut -d'.' -f1)
                IMG=${{ parameters.trivyImage }}
                REPO_TAG=${IMG#*/}
                if ! az acr repository show --name "$BASE_ACR" --image "$REPO_TAG" > /dev/null 2>&1; then
                  echo "Importing $REPO_TAG from Docker Hub"
                  az acr import --name "$BASE_ACR" --source docker.io/applabimg/$REPO_TAG --image $REPO_TAG || az acr repository show --name "$BASE_ACR" --image "$REPO_TAG"
                fi
                az acr login --name "$BASE_ACR"

          - bash: |
              set -e
              OUT=$(Build.ArtifactStagingDirectory)/sca
              mkdir -p $OUT
              FLAGS="--skip-db-update --offline-scan --skip-version-check --disable-telemetry --scanners vuln"
              docker run --rm --network none -v $(Agent.BuildDirectory)/deps:/src:ro -v $OUT:/out ${{ parameters.trivyImage }} fs $FLAGS --format sarif --output /out/trivy-deps.sarif /src
              docker run --rm --network none -v $(Agent.BuildDirectory)/deps:/src:ro ${{ parameters.trivyImage }} fs $FLAGS --severity ${{ parameters.severity }} --exit-code ${{ parameters.exitCode }} /src
            displayName: 'Trivy scan'

          - task: PublishBuildArtifacts@1
            displayName: 'Publish SCA results'
            condition: always()
            inputs:
              PathtoPublish: '$(Build.ArtifactStagingDirectory)/sca'
              ArtifactName: 'sca-sarif-results'
