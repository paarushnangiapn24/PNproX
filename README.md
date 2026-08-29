# PNproX
This web is a fast, encrypted, secure and private web. 
#!/usr/bin/env bash

set -euo pipefail


# publish_full.sh


# Single pasteable script to prepare PN24pro for publication and trigger CI.


# Save as publish_full.sh in the project root, make executable:


# chmod +x publish_full.sh


# Run:


# ./publish_full.sh GITHUB_OWNER REPO_NAME


# 


# Prereqs you run locally:


# - git configured and in project root (this script assumes repo files are present)


# - gh CLI installed and authenticated (gh auth login)


# - Optional: flutter installed for icon generation


# - Have signing files ready locally if you plan to upload them (google_play_service_account.json, android_keystore.jks, apple_api_key.p8)


# 


# This script does NOT accept or store tokens here — everything runs on your machine.


ROOT_DIR="$(cd "$(dirname "$0")" && pwd)"

cd "$ROOT_DIR"


if [ $# -lt 2 ]; then

echo "Usage: $0 GITHUB_OWNER REPO_NAME"

exit 2

fi

OWNER="$1"

REPO="$2"


echo "== PN24pro publish helper =="

echo "Working from: $ROOT_DIR"


# --- 1) generate platform icons (if script available) ---


if [ -x "./scripts/make_platform_icons.sh" ]; then

echo "[1/6] Generating platform icons..."

./scripts/make_platform_icons.sh || echo "Icon generation returned non-zero; continuing if icons exist."

else

echo "[1/6] make_platform_icons.sh not found/executable — skip. Ensure flutter_app/assets/icons/app_icon.png exists or generate icons manually."

fi


# --- 2) copy 1024 icon to Flutter asset if available ---


if [ -f "flutter_app/assets/icons/generated/icon_1024x1024.png" ]; then

echo "[2/6] Copying generated 1024 icon into flutter asset path..."

mkdir -p flutter_app/assets/icons

cp -f flutter_app/assets/icons/generated/icon_1024x1024.png flutter_app/assets/icons/app_icon.png

else

echo "[2/6] No generated 1024 icon found. If you have a source PNG, place it at flutter_app/assets/icons/app_icon.png"

fi


# --- 3) Run flutter_launcher_icons (if flutter installed) ---


if command -v flutter >/dev/null 2>&1; then

echo "[3/6] Running flutter_launcher_icons to generate platform icons..."

(cd flutter_app && flutter pub get && flutter pub run flutter_launcher_icons:main) || echo "flutter_launcher_icons failed; you can run it manually in flutter_app"

else

echo "[3/6] flutter CLI not found — skip. Install Flutter if you want this step automated."

fi


# --- 4) Create GitHub repo and push (requires gh auth) ---


if ! command -v gh >/dev/null 2>&1; then

echo "[4/6] gh CLI not found. Install gh: [https://cli.github.com/](https://cli.github.com/) and run 'gh auth login' then re-run this script."

exit 1

fi


echo "[4/6] Creating GitHub repo $OWNER/$REPO (if it doesn't exist) and pushing code..."

set +e

gh repo view "$OWNER/$REPO" >/dev/null 2>&1

EXISTS=$?

set -e

if [ $EXISTS -ne 0 ]; then

gh repo create "$OWNER/$REPO" --public --confirm || {

echo "Failed to create repo via gh. Create the repo manually and re-run push steps if needed."

}

else

echo "Repository $OWNER/$REPO already exists."

fi


# Add remote if missing


if ! git remote get-url origin >/dev/null 2>&1; then

git remote add origin "git@github.com:${OWNER}/${REPO}.git"

fi


echo "[4/6] Pushing local branch and tags to origin..."

git push -u origin "$(git rev-parse --abbrev-ref HEAD)" || true

git push --tags || true


# --- 5) Upload secrets interactively (via gh) ---


echo "[5/6] Uploading secrets to GitHub (interactive). Prepare your signing files if you will upload them."

read -p "Upload Google Play service-account JSON? (y/N) " RESP

if [[ "$RESP" =~ ^[Yy]$ ]]; then

read -p "Path to google_play_service_account.json: " GP_JSON

if [ -f "$GP_JSON" ]; then

gh secret set ANDROID_SERVICE_ACCOUNT_JSON -b < "$GP_JSON" --repo "$OWNER/$REPO"

echo "Uploaded ANDROID_SERVICE_ACCOUNT_JSON"

else

echo "File not found: $GP_JSON"

fi

fi


read -p "Upload Android keystore (.jks)? (y/N) " RESP

if [[ "$RESP" =~ ^[Yy]$ ]]; then

read -p "Path to android_keystore.jks: " KEYSTORE

if [ -f "$KEYSTORE" ]; then

# store binary keystore as base64 secret

if command -v base64 >/dev/null 2>&1; then

base64 "$KEYSTORE" | gh secret set ANDROID_KEYSTORE_BASE64 -b - --repo "$OWNER/$REPO"

else

echo "base64 unavailable; upload keystore manually in repo Settings -> Secrets."

fi

read -p "Android key alias (ANDROID_KEY_ALIAS): " ANDROID_KEY_ALIAS

gh secret set ANDROID_KEY_ALIAS --body "$ANDROID_KEY_ALIAS" --repo "$OWNER/$REPO"

read -s -p "Android key password (ANDROID_KEY_PASSWORD): " ANDROID_KEY_PASSWORD

echo

gh secret set ANDROID_KEY_PASSWORD --body "$ANDROID_KEY_PASSWORD" --repo "$OWNER/$REPO"

echo "Uploaded Android keystore secrets"

else

echo "File not found: $KEYSTORE"

fi

fi


read -p "Upload Apple App Store Connect key (.p8)? (y/N) " RESP

if [[ "$RESP" =~ ^[Yy]$ ]]; then

read -p "Path to App Store .p8 file: " APP_P8

if [ -f "$APP_P8" ]; then

gh secret set APP_STORE_CONNECT_KEY -b < "$APP_P8" --repo "$OWNER/$REPO"

read -p "App Store Issuer ID (APP_STORE_ISSUER_ID): " APP_STORE_ISSUER_ID

gh secret set APP_STORE_ISSUER_ID --body "$APP_STORE_ISSUER_ID" --repo "$OWNER/$REPO"

read -p "App Store Key ID (APP_STORE_KEY_ID): " APP_STORE_KEY_ID

gh secret set APP_STORE_KEY_ID --body "$APP_STORE_KEY_ID" --repo "$OWNER/$REPO"

read -p "Apple Team ID (APP_TEAM_ID): " APP_TEAM_ID

gh secret set APP_TEAM_ID --body "$APP_TEAM_ID" --repo "$OWNER/$REPO"

echo "Uploaded Apple App Store secrets"

else

echo "File not found: $APP_P8"

fi

fi


echo "[5/6] Secrets upload finished (or skipped). Verify Secrets in GitHub repo Settings if needed."


# --- 6) Create & push release tag to trigger CI ---


TAG="v0.1.0"

if git rev-parse "$TAG" >/dev/null 2>&1; then

echo "[6/6] Tag $TAG already exists locally."

else

echo "Creating annotated tag $TAG"

git tag -a "$TAG" -m "PN24pro $TAG"

fi


echo "Pushing tag $TAG to origin..."

git push origin "$TAG"


echo

echo "Done. CI workflows will run on GitHub. Monitor GitHub Actions for build progress and Releases for artifacts."

echo "Important next steps:"

echo " - Verify Actions -> release.yml ran and completed successfully."

echo " - Review generated artifacts and then submit to Google Play / App Store (metadata, screenshots, privacy policy)."

echo " - This script never sends your secrets to any external service other than GitHub via the gh CLI; ensure gh is authenticated locally (gh auth login)."


exit 0

















