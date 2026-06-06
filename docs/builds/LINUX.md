# Building Cake Wallet for Linux

## Requirements and Setup

The Linux build is fully Docker-based. The only dependency on your local host is the
Docker Engine.

- <https://docs.docker.com/engine/install/>

The builder image packages Flutter 3.32.8, Android NDK r28, Go 1.24.1, and a nightly Rust
toolchain. You do **not** need to install any of these manually.

> **Note (Apple Silicon):** If building on a Mac with an M-series CPU (arm64) you may
> encounter segmentation faults. Simply retry the build if this happens.

### External source dependencies

Several libraries are fetched at build time by prepare scripts (`scripts/prepare_*.sh`) and
built inside the container. They are **not** git submodules — they are cloned on demand into
`scripts/monero_c`, `scripts/torch_dart`, `scripts/zcash_lib`, `scripts/reown_flutter`, and
`scripts/bitbox_flutter`, all of which are gitignored. Do not create those directories
manually; the build scripts manage them.

---

## Building Cake Wallet or Monero.com

```bash
git clone --branch main https://github.com/cake-tech/cake_wallet.git
# NOTE: Replace `main` with the latest release tag:
#       https://github.com/cake-tech/cake_wallet/releases/latest
cd cake_wallet

# To build the Docker image yourself instead of pulling it, uncomment the next line:
# docker build -t ghcr.io/cake-tech/cake_wallet:debian13-flutter3.32.8-ndkr28-go1.24.1-ruststablenightly .

docker run --privileged -v$(pwd):$(pwd) -w $(pwd) -i --rm \
  ghcr.io/cake-tech/cake_wallet:debian13-flutter3.32.8-ndkr28-go1.24.1-ruststablenightly \
  bash -x << 'EOF'
set -x -e
git config --global --add safe.directory '*'
git config --global user.email "ci@cakewallet.com"
git config --global user.name "CakeWallet CI"

pushd scripts
    ./gen_android_manifest.sh
    ./prepare_moneroc.sh       # clones/pins scripts/monero_c
    ./prepare_torch.sh         # clones/pins scripts/torch_dart
    ./prepare_zcash.sh         # clones/pins scripts/zcash_lib
    ./prepare_reown.sh         # clones/pins scripts/reown_flutter
    ./build_bitbox_flutter.sh
    pushd android
        ./build_mwebd.sh       # builds MWEB daemon (requires Go)
    popd
popd

pushd scripts/linux
    ./build_monero_all.sh      # prepare_moneroc.sh already called above; safe to re-run
    ./build_zcash.sh
    source ./app_env.sh cakewallet
    # source ./app_env.sh monero.com  # Uncomment to build monero.com instead
    ./app_config.sh
popd

flutter pub get
flutter clean
./model_generator.sh
dart run tool/generate_localization.dart
dart run tool/generate_new_secrets.dart
flutter build linux --release

# Copy to a stable path (use arm64 line instead if building on ARM)
cp -r build/linux/x64/release/bundle build/linux/current

# Optional: build a Flatpak (requires --privileged Docker flag, already set above)
flatpak-builder --force-clean flatpak-build com.cakewallet.CakeWallet.yml
flatpak build-export export flatpak-build
flatpak build-bundle export build/linux/current/cake_wallet.flatpak com.cakewallet.CakeWallet
EOF
```

Expected output:

```
+ flutter build linux --release
Building Linux application...
✓ Built build/linux/x64/release/bundle/cake_wallet
```

The binary and its shared libraries are in `build/linux/x64/release/bundle/`.

### Installing the Flatpak

```bash
flatpak --user install build/linux/current/cake_wallet.flatpak
```
