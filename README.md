- bash: |
    set -e
    AUTH="AUTHORIZATION: bearer $(System.AccessToken)"
    git -c http.extraheader="$AUTH" fetch origin '+refs/heads/release/*:refs/remotes/origin/release/*'
    PREFIX="$(releaseMajor).$(releaseMinor)"
    LAST=$(git branch -r | grep "origin/release/$PREFIX\." | sed 's#.*/##' | sort -V | tail -1)
    if [ -z "$LAST" ]; then
      NEW=$PREFIX.1
    else
      NEW=${LAST%.*}.$(( ${LAST##*.} + 1 ))
    fi
    echo "Creating release/$NEW"
    git -c http.extraheader="$AUTH" push origin $(Build.SourceVersion):refs/heads/release/$NEW
  displayName: 'Create next release branch'
