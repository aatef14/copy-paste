- bash: |
    SHORT=$(echo $(Build.SourceVersion) | cut -c1-8)
    echo "##vso[task.setvariable variable=shortSha]$SHORT"
  displayName: 'Short commit ID'
