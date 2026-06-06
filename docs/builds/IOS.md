# Building Cake Wallet for iOS

## Requirements

```
macOS 15.3.1 or later
Xcode 16.2 or later
Flutter 3.32.8
Go 1.24.1 or later   (required by build_mwebd.sh)
Rust (stable + nightly, via rustup)
```

### External source dependencies

Several libraries are fetched at build time by prepare scripts (`scripts/prepare_*.sh`) and
built locally. They are **not** git submodules — they are cloned on demand into
`scripts/monero_c`, `scripts/torch_dart`, `scripts/zcash_lib`, `scripts/reown_flutter`, and
`scripts/bitbox_flutter`, all of which are gitignored. Do not create those directories
manually; the build scripts manage them.

---

## Step-by-step setup

### 1. Install system dependencies

Install [Homebrew](https://brew.sh) if you haven't already, then:

```zsh
brew install automake ccache cmake cocoapods go libtool pkg-config xz
sudo softwareupdate --install-rosetta --agree-to-license
```

### 2. Install Xcode

Download and install Xcode from the macOS App Store, then initialize it:

```zsh
sudo xcode-select --switch /Applications/Xcode.app/Contents/Developer
sudo xcodebuild -runFirstLaunch
```

To enable iOS build support:

1. Open Xcode → Settings → Components
2. Click **Get** next to the latest iOS simulator runtime

### 3. Install Flutter 3.32.8

Download Flutter 3.32.8 from the [Flutter release archive](https://docs.flutter.dev/release/archive)
(the main install page links to the latest version, which may differ).

Add Flutter to your `PATH` as described in the official docs.

### 4. Install Rust

```zsh
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source "$HOME/.cargo/env"

# Add required iOS targets
rustup target add aarch64-apple-ios
rustup target add aarch64-apple-ios-sim
rustup target add x86_64-apple-ios
```

### 5. Verify Flutter and Xcode

```zsh
flutter doctor
```

Required output:

```
[✓] Flutter (Channel stable, 3.32.8, on macOS 15.x.x ...)
[✓] Xcode - develop for iOS and macOS (Xcode 16.x)
```

Resolve any reported issues before continuing.

### 6. Clone the repository

```zsh
git clone https://github.com/cake-tech/cake_wallet.git --branch main
# NOTE: Replace `main` with the latest release tag:
#       https://github.com/cake-tech/cake_wallet/releases/latest
cd cake_wallet/scripts/ios/
```

### 7. Configure the app variant

```zsh
source ./app_env.sh cakewallet
# For Monero.com:  source ./app_env.sh monero.com
```

### 8. Build native libraries

Each script below fetches its own external dependency (via the `prepare_*.sh` scripts at
`scripts/`) before building.

```zsh
./build_monero_all.sh   # fetches and builds monero_c (monero, wownero, zano)
./build_mwebd.sh        # builds the MWEB daemon (requires Go)
./build_decred.sh       # builds the Decred libwallet
./build_zcash.sh        # fetches and builds zcash_lib
```

> **Note:** This step takes a long time. Build artifacts are cached in
> `cw_monero/ios/External/`, `cw_mweb/ios/`, etc.

Then apply app name, icon, and bundle ID:

```zsh
./app_config.sh
```

### 9. Install Flutter dependencies and generate code

```zsh
cd ../../   # return to repo root

flutter pub get
dart run tool/generate_new_secrets.dart
dart run tool/generate_localization.dart
./model_generator.sh
```

### 10. Build

```zsh
flutter build ios --release --no-codesign
```

Open `ios/Runner.xcworkspace` in Xcode to archive and sign the application.

To run directly on a connected device:

```zsh
flutter run
```
