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
