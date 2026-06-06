# Building Cake Wallet for Windows

## Requirements

```
Windows 10 or later (64-bit, x86-64)
Flutter 3.32.8
Visual Studio 2022 with "Desktop Development with C++" workload
Git for Windows
WSL 2 (Ubuntu) with MinGW cross-compiler   (required to build Monero dependencies)
Go 1.24.1 or later (inside WSL)
Rust (inside WSL, via rustup)
```

---

## Step-by-step setup

### 1. Install Flutter 3.32.8

Download Flutter 3.32.8 from the [Flutter release archive](https://docs.flutter.dev/release/archive)
(the main install page links to the latest version, which may differ) and follow the
[Windows installation docs](https://docs.flutter.dev/get-started/install/windows).

Enable Developer Mode: Start → Run → `ms-settings:developers` → turn on **Developer Mode**.

### 2. Install Visual Studio 2022 and Git

Follow the [Flutter Development Tools](https://docs.flutter.dev/get-started/install/windows/desktop#development-tools)
instructions. Be sure to select the **Desktop Development with C++** workload in the VS
installer.

Add Git to your `PATH`:
Start → "environment" → Environment Variables → double-click **Path** →
add `C:\Program Files\Git\bin\` on a new line.

Install NuGet separately:

1. Download `nuget.exe` from <https://dist.nuget.org/win-x86-commandline/latest/nuget.exe>
2. Create `C:\Program Files\Nuget\` and move `nuget.exe` there.
3. Add `C:\Program Files\Nuget\` to your `PATH` as above.

### 3. Install WSL 2 (Ubuntu)

Open PowerShell as Administrator and run:

```powershell
wsl --install
```

Then install the build dependencies inside WSL:

```powershell
wsl sudo apt update
wsl sudo apt install -y autoconf build-essential ccache cmake curl gcc \
    gcc-mingw-w64-x86-64 git g++ g++-mingw-w64-x86-64 gperf lbzip2 \
    libtool make pkg-config pigz
```

### 4. Install Go and Rust inside WSL

```bash
# Go
wsl bash -c "wget https://go.dev/dl/go1.24.1.linux-amd64.tar.gz && \
    sudo rm -rf /usr/local/go && \
    sudo tar -C /usr/local -xzf go1.24.1.linux-amd64.tar.gz && \
    rm go1.24.1.linux-amd64.tar.gz && \
    echo 'export PATH=\$PATH:/usr/local/go/bin' >> ~/.bashrc"

# Rust
wsl curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | wsl sh
```

### 5. Clone the repository

```powershell
git clone https://github.com/cake-tech/cake_wallet.git --branch main
# NOTE: Replace `main` with the latest release tag:
#       https://github.com/cake-tech/cake_wallet/releases/latest
cd cake_wallet
```

### 6. Build Monero and dependencies (inside WSL)

The Monero C library must be cross-compiled for Windows using MinGW inside WSL.
Run the following from a WSL shell **while your working directory is the repo root**:

```bash
wsl
git config --global user.email "builds@cakewallet.com"
git config --global user.name "builds"
cd scripts/windows
./build_all.sh
```

`build_all.sh` will:
1. Clone/pin `scripts/monero_c` via `prepare_moneroc.sh`
2. Cross-compile `monero` and `wownero` for `x86_64-w64-mingw32`
3. Decompress the resulting `.dll.xz` files into place

When done, exit WSL:

```bash
exit
```

### 7. Configure and build the application

From PowerShell in the repo root:

```powershell
.\cakewallet.bat
```

The script will:
1. Generate `pubspec.yaml` from `pubspec_description.yaml`
2. Run `dart run tool/configure.dart` with all supported coin flags
3. Generate placeholder API secrets (`lib/.secrets.g.dart`) if not already present
4. Generate MobX models and localization files
5. Run `flutter build windows --release`
6. Copy the required Visual C++ runtime DLLs into the output directory
7. Produce `Cake Wallet.zip` in the repo root

Extract the zip and run `CakeWallet.exe`.

---

## Notes

- The `cakewallet.bat` script expects `bash.exe` in `PATH` (provided by Git for Windows).
- To override the Visual C++ runtime path (if VS is not installed to the default location),
  pass it as the first argument: `.\cakewallet.bat "C:\path\to\VC\Redist\..."`.
- If you already have `lib\.secrets.g.dart` with real API keys, the bat will not overwrite it.
