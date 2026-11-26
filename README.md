trigger:
  branches:
    include:
      - release-branch

pr: none

name: 'release-$(Rev:r)'

variables:
  - group: vg-qisp-bridge-release

stages:
  - stage: Release_Branch
    displayName: 'Create release branch'
    jobs:
      - job: Create
        displayName: 'create new release'
        pool:
          name: '$(poolName)'
        steps:
          - checkout: self
            fetchDepth: 0
            clean: true
          - bash: |
              set -e
              AUTH="AUTHORIZATION: bearer $(System.AccessToken)"
              git -c http.extraheader="$AUTH" fetch --prune origin '+refs/heads/release/*:refs/remotes/origin/release/*'
              PREFIX="$(releaseMajor).$(releaseMinor)"
              LAST=$(git branch -r | grep -E "origin/release/(archive/)?$PREFIX\." | sed 's#.*/##' | sort -V | tail -1)
              if [ -z "$LAST" ]; then
                NEW=$PREFIX.1
              else
                NEW=${LAST%.*}.$(( ${LAST##*.} + 1 ))
              fi
              OLD=$(git branch -r | grep "origin/release/" | grep -v "origin/release/archive/" | sed 's#.*origin/release/##')
              echo "Creating release/$NEW"
              git -c http.extraheader="$AUTH" push origin $(Build.SourceVersion):refs/heads/release/$NEW
              for V in $OLD; do
                echo "Archiving release/$V -> release/archive/$V"
                git -c http.extraheader="$AUTH" push origin refs/remotes/origin/release/$V:refs/heads/release/archive/$V
                git -c http.extraheader="$AUTH" push origin --delete release/$V
              done
            displayName: 'Create next release branch and archive old ones'
