trigger: none

pr: none

name: 'Trivy-Test-$(Build.BuildId)'

parameters:
  - name: testImage
    displayName: 'Existing image in dev ACR to scan'
    type: string
    default: 'frontend-dev:66053412-52484'
  - name: exitCode
    displayName: 'Fail on findings? (0 = report only, 1 = fail)'
    type: string
    default: '0'

resources:
  repositories:
    - repository: deps
      type: git
      name: ITS21/bridge-api-mgmt-portal
      ref: refs/heads/dependency

variables:
  - group: vg-qisp-bridge-dev
  - name: trivyImage
    value: 'qispnpacr01.azurecr.io/trivy-offline:0.69.3-2026-11-19'

stages:
  - template: templates/stage-dependency-scan.yml
    parameters:
      poolName: '$(poolName)'
      baseServiceConnection: '$(baseServiceConnectionName)'
      baseAcrLoginServer: '$(baseAcrLoginServer)'
      trivyImage: '$(trivyImage)'
      exitCode: '${{ parameters.exitCode }}'

  - stage: ImageScan
    displayName: 'Image Scan (Trivy)'
    dependsOn: []
    jobs:
      - job: Trivy
        displayName: 'Scan ${{ parameters.testImage }}'
        pool:
          name: '$(poolName)'
        steps:
          - checkout: none

          - task: AzureCLI@2
            displayName: 'Login to dev ACR'
            inputs:
              azureSubscription: '$(baseServiceConnectionName)'
              scriptType: bash
              scriptLocation: inlineScript
              inlineScript: |
                set -e
                ACR=$(echo $(baseAcrLoginServer) | cut -d'.' -f1)
                IMG=$(trivyImage)
                REPO_TAG=${IMG#*/}
                if ! az acr repository show --name "$ACR" --image "$REPO_TAG" > /dev/null 2>&1; then
                  echo "Importing $REPO_TAG from Docker Hub"
                  az acr import --name "$ACR" --source docker.io/applabimg/$REPO_TAG --image $REPO_TAG || az acr repository show --name "$ACR" --image "$REPO_TAG"
                fi
                az acr login --name "$ACR"

          - bash: |
              set -e
              IMG=$(baseAcrLoginServer)/${{ parameters.testImage }}
              docker pull $IMG
              docker run --rm --network none -v /var/run/docker.sock:/var/run/docker.sock $(trivyImage) image --skip-db-update --offline-scan --skip-version-check --disable-telemetry --scanners vuln --severity CRITICAL,HIGH --exit-code ${{ parameters.exitCode }} $IMG
            displayName: 'Trivy image scan'

          - bash: |
              docker rmi $(baseAcrLoginServer)/${{ parameters.testImage }} || true
            displayName: 'Remove scanned image'
            condition: always()
