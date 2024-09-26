# GitHub Action: GitHub-action-release-apk-expo

## About

This GitHub Action automates the process of building an Android APK for an Expo project whenever a new tag is created. It generates a release from the tag and attaches the built APK to the release assets, ensuring that each version has a downloadable APK directly linked to its release.

## Key Features:

- Automated APK builds: Automatically compiles your APK whenever a new tag is pushed.

- Tag-based Releases: Generates a new GitHub release for each tag.

- APK in Release Assets: Attaches the built APK to the newly created release for easy access.
  
- Expo EAS Integration: Leverages Expo's EAS (Expo Application Services) to build the APK seamlessly.

## Installation
  1. Add Workflow File:
Add the action.yml as follows .github/workflows/build.yml in the project. For more reference see the [official docs](https://help.github.com/en/actions/configuring-and-managing-workflows/configuring-a-workflow#creating-a-workflow-file)

```yaml
name: Android APK Build & Release (Tag Or Manual Trigger)

on:
  push:
    tags:
      - 'v*' # Trigger on tag push that starts with 'v'
  workflow_dispatch: # Allow manual trigger for testing

permissions:
  contents: write # Grant permissions to create and upload releases
  issues: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Setup Android SDK
        uses: android-actions/setup-android@v3

      - name: Cache Gradle packages
        uses: actions/cache@v3
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
          restore-keys: |
            ${{ runner.os }}-gradle-
            ${{ runner.os }}-gradle-default

      - name: Cache EAS build
        uses: actions/cache@v3
        with:
          path: ~/.eas-cli
          key: ${{ runner.os }}-eas-${{ hashFiles('**/eas.json') }}
          restore-keys: |
            ${{ runner.os }}-eas-
            ${{ runner.os }}-eas-default

      - name: Setup Expo
        uses: expo/expo-github-action@v8
        with:
          expo-version: latest
          eas-version: latest
          token: ${{ secrets.EXPO_TOKEN }}

      - name: Build Android app
        run: |
          npx eas-cli build --platform android \
            --profile production \
            --local --non-interactive \
            --output app-production-release.apk

      - name: Upload APK artifact
        uses: actions/upload-artifact@v4
        with:
          name: app-release
          path: app-production-release.apk

      - name: Create Release and Upload APK
        if: startsWith(github.ref, 'refs/tags/')
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          TAG_NAME=${GITHUB_REF#refs/tags/}
          RELEASE_NAME="Release $TAG_NAME"
          if ! gh release view $TAG_NAME > /dev/null 2>&1; then
            gh release create "$TAG_NAME" app-production-release.apk \
              --title "$RELEASE_NAME" \
              --notes "APK for version $TAG_NAME"
          else
            gh release upload "$TAG_NAME" app-production-release.apk --clobber
          fi

```

  2. Secrets Configuration:

Add your EXPO_TOKEN to your repository's secrets.
  - First get the EXPO_TOKEN from expo.dev website ([officiel docs](https://docs.expo.dev/accounts/programmatic-access/)) 
  - Add the secret to your GitHub repository using the ([officiel docs](https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions)).
You dont need to deal with GITHUB_TOKEN, its managed by GitHub.

## Usage

- **Trigger by Tags:** Push a new tag (e.g., v1.0.0) to automatically build the APK and generate a release.

```bash
git tag -a "v1.0.0" -m "First release"
git push origin v1.0.0
```

- **Manual Trigger:** You can also trigger the workflow manually using the "Run workflow" button in the GitHub Actions tab.

## Troubleshoot

- **APK Not Attached to Release:**

  Ensure that the GITHUB_TOKEN secret is correctly set and has the necessary permissions (contents: write).

- **Expo Build Fails:**

  Make sure that the Expo CLI and EAS CLI versions are up to date.
  Check if your eas.json file is correctly configured and that the production build profile is defined.

- **Gradle Cache Not Working:**

  If Gradle caching fails or isn't working as expected, ensure that the Gradle files (build.gradle, gradle-wrapper.properties) are properly hashed in the cache step.

- **Build Timeout:**

  For larger apps or heavy builds, consider increasing the job timeout using the timeout-minutes option under jobs.build.

## Contact Us

For any questions, issues, or feedback, feel free to reach out:

 - **GitHub Issues:** Open an issue in the ([GitHub repository](https://www.github.com/adelpro/GitHub-action-release-apk/issues)).
 - **Twitter:** Reach us on Twitter @adelpro.

We are happy to assist with any issues or feedback you may have regarding this GitHub Action.

