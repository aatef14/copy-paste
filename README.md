          - bash: |
              SHORT=$(echo $(Build.SourceVersion) | cut -c1-8)
              echo "##vso[task.setvariable variable=shortSha]$SHORT"
              echo "##vso[build.updatebuildnumber]$SHORT"
            displayName: 'Short commit ID'
