# Wingsma
Ai keyboard
name: Build Wingman APK

on:
  push:
    paths:
      - "wingman-project.zip"
  workflow_dispatch:

permissions:
  contents: write

jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v4

      - name: Unpack project (when a zip was uploaded)
        run: |
          if [ -f wingman-project.zip ]; then
            unzip -q wingman-project.zip -d /tmp/unz
            cp -a /tmp/unz/Wingman/. .
            git rm -q -f wingman-project.zip
            echo "UNPACKED=1" >> "$GITHUB_ENV"
          fi

      - name: Save the unpacked source in the repo
        if: env.UNPACKED == '1'
        continue-on-error: true
        run: |
          git config user.name "github-actions"
          git config user.email "github-actions@users.noreply.github.com"
          git add -A
          git commit -q -m "Unpack Wingman project"
          git push

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "17"

      - uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: "8.7"

      - name: Build debug APK
        env:
          BACKEND_URL: ${{ vars.BACKEND_URL }}
        run: |
          EXTRA=""
          if [ -n "$BACKEND_URL" ]; then EXTRA="-PbackendUrl=$BACKEND_URL"; fi
          gradle assembleDebug --stacktrace --console=plain $EXTRA

      - name: Collect APK
        run: cp app/build/outputs/apk/debug/app-debug.apk Wingman.apk

      - uses: actions/upload-artifact@v4
        with:
          name: Wingman-apk
          path: Wingman.apk

      - name: Publish release
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          gh release create "build-${{ github.run_number }}" Wingman.apk \
            --title "Wingman build ${{ github.run_number }}" \
            --notes "Debug APK. Download Wingman.apk on your phone and install it."

