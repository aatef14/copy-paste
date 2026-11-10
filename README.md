trigger:
  branches:
    include:
      - main

pr: none

name: 'prod-$(Rev:r)'

parameters:
  - name: components
    type: object
    default:
      - key: frontend
        sourceImage: '$(sourceImageFrontend)'
        image: '$(imageFrontend)'
        appService: '$(devAppServiceName)'
      - key: backend
        sourceImage: '$(sourceImageBackend)'
        image: '$(imageBackend)'
        appService: '$(devCronAppServiceName)'

variables:
  - group: vg-qisp-bridge-prod

stages:
  - stage: Promote_Prod
    displayName: 'Promote tested images to Prod'
    jobs:
      - deployment: Promote
        displayName: 'Promote and deploy'
        environment: '$(environmentName)'
        pool:
          name: '$(poolName)'
        strategy:
          runOnce:
            deploy:
              steps:
                - checkout: self
                  clean: true
                  fetchDepth: 2

                - bash: |
                    set -e
                    SHA=$(git rev-parse HEAD^2) || { echo "##vso[task.logissue type=error]main commit is not a merge commit"; exit 1; }
                    AUTH="AUTHORIZATION: bearer $(System.AccessToken)"
                    VER=$(git -c http.extraheader="$AUTH" ls-remote origin 'refs/heads/release/*' | grep "^$SHA" | sed 's#.*refs/heads/release/##' | sort -V | tail -1)
                    if [ -z "$VER" ]; then
                      echo "##vso[task.logissue type=error]No release branch points at $SHA"
                      exit 1
                    fi
                    echo "Promoting commit $SHA as v$VER"
                    echo "##vso[task.setvariable variable=sourceTag]$SHA"
                    echo "##vso[task.setvariable variable=releaseVersion]v$VER"
                    echo "##vso[build.updatebuildnumber]v$VER"
                  displayName: 'Find release version'

                - ${{ each c in parameters.components }}:
                  - task: AzureCLI@2
                    displayName: 'Import ${{ c.key }} image'
                    inputs:
                      azureSubscription: '$(azureServiceConnectionName)'
                      scriptType: bash
                      scriptLocation: inlineScript
                      inlineScript: |
                        ACR_NAME=$(echo $(acrLoginServer) | cut -d'.' -f1)
                        az acr import --name "$ACR_NAME" --registry "$(sourceAcrResourceId)" --source ${{ c.sourceImage }}:$(sourceTag) --image ${{ c.image }}:$(releaseVersion)

                - ${{ each c in parameters.components }}:
                  - task: AzureWebAppContainer@1
                    displayName: 'Deploy ${{ c.key }} (Prod)'
                    inputs:
                      azureSubscription: '$(azureServiceConnectionName)'
                      appName: '${{ c.appService }}'
                      containers: '$(acrLoginServer)/${{ c.image }}:$(releaseVersion)'

                - bash: |
                    set -e
                    git tag $(releaseVersion) HEAD
                    git -c http.extraheader="AUTHORIZATION: bearer $(System.AccessToken)" push origin $(releaseVersion)
                  displayName: 'Create git tag'
