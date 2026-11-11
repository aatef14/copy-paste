trigger:
  branches:
    include:
      - main

pr: none

name: 'bridge-prod-$(BuildID)'

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
                    SHORT=$(echo $SHA | cut -c1-8)
                    echo "Promoting commit $SHORT as v$VER"
                    echo "##vso[task.setvariable variable=sourceTag]$SHORT"
                    echo "##vso[task.setvariable variable=releaseVersion]v$VER"
                    echo "##vso[build.updatebuildnumber]bridge-prod-v$VER"
                  displayName: 'Find release version'

                - task: AzureCLI@2
                  displayName: 'Login to Test ACR'
                  inputs:
                    azureSubscription: '$(sourceServiceConnectionName)'
                    scriptType: bash
                    scriptLocation: inlineScript
                    inlineScript: |
                      set -e
                      SRC_ACR=$(echo $(sourceAcrLoginServer) | cut -d'.' -f1)
                      az acr login --name "$SRC_ACR"
                      TAG=$(az acr repository show-tags --name "$SRC_ACR" --repository $(sourceImageFrontend) --query "[?starts_with(@, '$(sourceTag)-')]" -o tsv | sort -t- -k2 -n | tail -1)
                      if [ -z "$TAG" ]; then
                        echo "##vso[task.logissue type=error]No test image found for commit $(sourceTag)"
                        exit 1
                      fi
                      echo "Found test image tag $TAG"
                      echo "##vso[task.setvariable variable=sourceTag]$TAG"

                - ${{ each c in parameters.components }}:
                  - task: AzureCLI@2
                    displayName: 'Pull, tag and push ${{ c.key }}'
                    inputs:
                      azureSubscription: '$(azureServiceConnectionName)'
                      scriptType: bash
                      scriptLocation: inlineScript
                      inlineScript: |
                        set -e
                        ACR_NAME=$(echo $(acrLoginServer) | cut -d'.' -f1)
                        az acr login --name "$ACR_NAME"
                        SRC=$(sourceAcrLoginServer)/${{ c.sourceImage }}:$(sourceTag)
                        DST=$(acrLoginServer)/${{ c.image }}:$(sourceTag)
                        docker pull $SRC
                        docker tag $SRC $DST
                        docker push $DST
                        docker rmi $SRC $DST

                - ${{ each c in parameters.components }}:
                  - task: AzureWebAppContainer@1
                    displayName: 'Deploy ${{ c.key }} (Prod)'
                    inputs:
                      azureSubscription: '$(azureServiceConnectionName)'
                      appName: '${{ c.appService }}'
                      containers: '$(acrLoginServer)/${{ c.image }}:$(sourceTag)'

                - bash: |
                    set -e
                    git tag $(releaseVersion) HEAD
                    git -c http.extraheader="AUTHORIZATION: bearer $(System.AccessToken)" push origin $(releaseVersion)
                  displayName: 'Create git tag'
