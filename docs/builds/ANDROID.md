# Building Cake Wallet for Android

## Requirements and Setup

The Android build is fully Docker-based. The only dependency on your local host is the
Docker Engine.

- <https://docs.docker.com/engine/install/>

The builder image packages Flutter 3.32.8, Android NDK r28, Go 1.24.1, and a nightly Rust
toolchain. You do **not** need to install any of these manually.

### External source dependencies

Several libraries are fetched at build time by prepare scripts (`scripts/prepare_*.sh`) and
built inside the container. They are **not** git submodules — they are cloned on demand into
`scripts/monero_c`, `scripts/torch_dart`, `scripts/zcash_lib`, `scripts/reown_flutter`, and
`scripts/bitbox_flutter`, all of which are gitignored. Do not create those directories
manually; the build scripts manage them.

---

## Building Cake Wallet or Monero.com

### Using the pre-built builder image

```bash
git clone --branch main https://github.com/cake-tech/cake_wallet.git
# NOTE: Replace `main` with the latest release tag:
#       https://github.com/cake-tech/cake_wallet/releases/latest
cd cake_wallet

# To build the Docker image yourself instead of pulling it, uncomment the next line:
# docker build -t ghcr.io/cake-tech/cake_wallet:debian13-flutter3.32.8-ndkr28-go1.24.1-ruststablenightly .

docker run -v$(pwd):$(pwd) -w $(pwd) -i --rm \
  ghcr.io/cake-tech/cake_wallet:debian13-flutter3.32.8-ndkr28-go1.24.1-ruststablenightly \
  bash -x << 'EOF'
set -x -e
git config --global --add safe.directory '*'

pushd scripts/android
    # Optional: builds embedded Tor support.
    # Will fail on case-insensitive filesystems and adds a few hours to the build.
    # ./build_torch.sh

    ./build_reown_deps.sh
    pushd ..
        ./build_bitbox_flutter.sh
    popd
    source ./app_env.sh cakewallet
    # source ./app_env.sh monero.com  # Uncomment to build monero.com instead
    ./app_config.sh
    ./build_monero_all.sh   # Calls scripts/prepare_moneroc.sh internally
    ./build_decred.sh
    ./build_mwebd.sh
    ./build_zcash.sh        # Calls scripts/prepare_zcash.sh internally
popd

# Generate a self-signed debug keystore if one doesn't exist yet
pushd android/app
    [[ -f key.jks ]] || keytool -genkey -v \
        -keystore key.jks -keyalg RSA -keysize 2048 -validity 10000 \
        -alias testKey -noprompt \
        -dname "CN=CakeWallet, OU=CakeWallet, O=CakeWallet, L=Florida, S=America, C=USA" \
        -storepass hunter1 -keypass hunter1
popd

flutter pub get
flutter clean
./model_generator.sh
dart run tool/generate_android_key_properties.dart \
    keyAlias=testKey storeFile=key.jks storePassword=hunter1 keyPassword=hunter1
dart run tool/generate_localization.dart
dart run tool/generate_new_secrets.dart
flutter build apk --release --split-per-abi
EOF
```

Expected output:

```
Running Gradle task 'assembleRelease'...                          519.1s
✓ Built build/app/outputs/flutter-apk/app-armeabi-v7a-release.apk (56.3MB)
✓ Built build/app/outputs/flutter-apk/app-arm64-v8a-release.apk (55.8MB)
✓ Built build/app/outputs/flutter-apk/app-x86_64-release.apk (56.4MB)
```

Final APKs are in `build/app/outputs/flutter-apk/`.

---

## Signing builds

Properly signing release APKs is outside the scope of this guide. See the Zeus team's
reproducible-builds guide for details:

- <https://github.com/ZeusLN/zeus/blob/master/docs/ReproducibleBuilds.md#signing-apks>
