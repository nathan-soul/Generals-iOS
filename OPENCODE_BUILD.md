# 🍎 Generals Zero Hour iOS — OpenCode Build Prompt

## Doel
Clone, build en sideload **Command & Conquer: Generals Zero Hour** naar een iPhone (iOS 17+). Volledig lokaal vanaf deze Mac, met game assets.

---

## 1. Vereisten check

```bash
xcode-select -p                    # moet Xcode 16.2+ tonen
brew --version                     # Homebrew geïnstalleerd?
git --version
```

- **macOS** Sequoia 15.0+
- **Xcode 16.2+** (via Mac App Store of Xcodes.app)
- **Homebrew**: `/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"`
- **iPhone** met iOS 17+, USB kabel, **AltStore** geïnstalleerd (via AltServer op deze Mac)

---

## 2. Repo clonen

```bash
git clone --recursive https://github.com/nathan-soul/Generals-iOS.git
cd Generals-iOS
```

> ⚠️ `--recursive` is cruciaal — anders missen alle submodules (DXVK, openal, etc.)

---

## 3. Systeem dependencies

```bash
brew install cmake ninja meson pkgconf xcodegen glslang
```

---

## 4. Vulkan SDK + MoltenVK

```bash
# Vulkan SDK 1.4.309.0
curl -sL https://sdk.lunarg.com/sdk/download/1.4.309.0/mac/vulkansdk-macos-1.4.309.0.zip -o /tmp/vulkan.zip
mkdir -p ~/VulkanSDK
unzip -q /tmp/vulkan.zip -d ~/VulkanSDK/
export VULKAN_SDK=~/VulkanSDK/macOS

# MoltenVK iOS static lib
./scripts/build/ios/fetch-moltenvk.sh

# Koppel MoltenVK aan VULKAN_SDK zodat CMake het vindt
MVK_STATIC="$HOME/GeneralsX/MoltenVK/MoltenVK/MoltenVK/static/MoltenVK.xcframework/ios-arm64/libMoltenVK.a"
mkdir -p "$VULKAN_SDK/lib/MoltenVK.xcframework/ios-arm64"
cp "$MVK_STATIC" "$VULKAN_SDK/lib/MoltenVK.xcframework/ios-arm64/libMoltenVK.a"
```

---

## 5. vcpkg

```bash
git clone https://github.com/microsoft/vcpkg ~/vcpkg
~/vcpkg/bootstrap-vcpkg.sh -disableMetrics
export VCPKG_ROOT=~/vcpkg
```

> Voeg `export VCPKG_ROOT=~/vcpkg` toe aan `~/.zshrc` voor persistentie.

---

## 6. Fonts

```bash
./scripts/build/ios/stage-fonts.sh
```

---

## 7. Game assets

**Optie A — Steam (aanbevolen):**
```bash
# Vind Steam library map
find ~/Library/Application\ Support/Steam -name "Generals.exe" -path "*Zero Hour*" 2>/dev/null

# Als gevonden, kopieer de hele data map
# (vervang pad met wat find oplevert)
cp -R "/path/to/Command and Conquer Generals Zero Hour/" ~/GeneralsX/GeneralsZH/
```

**Optie B — direct download** (als je de assets al hebt):
```bash
mkdir -p ~/GeneralsX/GeneralsZH
cp -R /pad/naar/GameData/* ~/GeneralsX/GeneralsZH/
```

**Wat er moet in zitten** (check):
```bash
ls ~/GeneralsX/GeneralsZH/*.big | head -10
# Verwachte bestanden: INI.big, Audio.big, Textures.big, W3D.big, Music.big, etc.
```

---

## 8. CMake configureren

```bash
cmake --preset ios-vulkan -DCMAKE_OSX_DEPLOYMENT_TARGET=17.0
```

✅ **Als dit faalt**: check of `VULKAN_SDK` en `VCPKG_ROOT` correct zijn geëxporteerd.

---

## 9. Engine bouwen

```bash
cmake --build build/ios-vulkan --target z_generals -j$(sysctl -n hw.ncpu)
```

Duurt ~5-10 minuten. Als die klaar is staat het binary in:
`build/ios-vulkan/GeneralsMD/GeneralsXZH.app/GeneralsXZH`

---

## 10. .ipa maken (unsigned, met alle dylibs)

```bash
BUILD_DIR="build/ios-vulkan"
APP_DIR="${BUILD_DIR}/GeneralsMD/GeneralsXZH.app"
DXVK_BUILD="${BUILD_DIR}/_deps/dxvk-build-macos"

# Maak Frameworks directory
mkdir -p "${APP_DIR}/Frameworks"

# Kopieer runtime dylibs
for lib in \
  "${DXVK_BUILD}/src/d3d8/libdxvk_d3d8.0.dylib" \
  "${DXVK_BUILD}/src/d3d9/libdxvk_d3d9.0.dylib" \
  "${BUILD_DIR}/_deps/sdl3-build/libSDL3.0.dylib" \
  "${BUILD_DIR}/_deps/sdl3_image-build/libSDL3_image.0.dylib" \
  "${BUILD_DIR}/_deps/openal_soft-build/libopenal.1.24.2.dylib" \
  "${BUILD_DIR}/libgamespy.dylib"; do
  [[ -f "$lib" ]] && cp "$lib" "${APP_DIR}/Frameworks/" && echo "  embedded $(basename $lib)"
done

# OpenAL hernoemen naar dyld-verwachte naam
[[ -f "${APP_DIR}/Frameworks/libopenal.1.24.2.dylib" ]] && \
  mv "${APP_DIR}/Frameworks/libopenal.1.24.2.dylib" "${APP_DIR}/Frameworks/libopenal.1.dylib"

# MoltenVK.framework (DXVK dlopen't deze bij Vulkan init)
MVK_FW="$HOME/GeneralsX/MoltenVK/MoltenVK/MoltenVK/dynamic/MoltenVK.xcframework/ios-arm64/MoltenVK.framework"
if [[ -d "$MVK_FW" ]]; then
  cp -R "$MVK_FW" "${APP_DIR}/Frameworks/"
  echo "  embedded MoltenVK.framework"
else
  echo "ERROR: MoltenVK.framework niet gevonden!"
  exit 1
fi

# rpath toevoegen zodat dyld Frameworks/ vindt
install_name_tool -add_rpath "@executable_path/Frameworks" "${APP_DIR}/GeneralsXZH" 2>/dev/null || true

# Game assets bundelen
mkdir -p "${APP_DIR}/GameData"
rsync -a --exclude=".*" \
  --exclude="*.dylib" --exclude="run.sh" --exclude="GeneralsXZH" \
  --exclude="*.DLL" --exclude="*.dll" --exclude="*.dat" --exclude="*.ico" \
  --exclude="*.bmp" --exclude="*.doc" --exclude="*.lcf" --exclude="Launcher.txt" \
  --exclude="MSS" --exclude="Manuals" --exclude="steamapps" \
  --exclude="steam_appid.txt" --exclude="00000000.*" \
  --exclude="RedistInstallers" --exclude="_CommonRedist" --exclude="*.txt" \
  ~/GeneralsX/GeneralsZH/ "${APP_DIR}/GameData/"

echo "GameData: $(du -sh "${APP_DIR}/GameData" | cut -f1)"

# Zip als .ipa
cd "$BUILD_DIR"
rm -f GeneralsZH-iOS.ipa
mkdir -p Payload
cp -R "${APP_DIR}" Payload/
cd Payload
zip -q -r --symlinks ../GeneralsZH-iOS.ipa GeneralsXZH.app
cd ..
rm -rf Payload
echo ".ipa created: $(ls -lh GeneralsZH-iOS.ipa | awk '{print $5}')"
```

✅ **.ipa klaar** in `build/ios-vulkan/GeneralsZH-iOS.ipa`

---

## 11. Naar iPhone krijgen

### Optie A — AltStore (aanbevolen, gratis)
```bash
open build/ios-vulkan/GeneralsZH-iOS.ipa
```
→ AltStore opent automatisch → **kies je iPhone** → wacht op installatie

### Optie B — Finder (ouderwets)
1. Sluit iPhone aan via USB
2. Open **Finder** → iPhone → **Apps**
3. Sleep `.ipa` naar het apps-venster
4. Klik **Sync**

### Optie C — devicectl (alleen met Apple Developer account)
```bash
xcrun devicectl device install app --device "$(xcrun devicectl list devices 2>/dev/null | grep -i connected | grep -oE '[0-9A-F-]{36}' | head -1)" "${APP_DIR}"
```

---

## 12. Eerste keer opstarten

1. Instellingen → **Algemeen** → **VPN & Apparaatbeheer**
2. Tik op het **Apple Developer** certificaat
3. Tik **Vertrouw**
4. Start **GeneralsZH** 🎮

---

## Veelvoorkomende fouten

| Fout | Oorzaak | Oplossing |
|---|---|---|
| `dyld: Library not loaded: @rpath/libSDL3.0.dylib` | Dylib mist in Frameworks/ | Check stap 10 — kopieert elke dylib? |
| `App name invalid` (AltStore) | Info.plist mist CFBundleIdentifier | Repo heeft deze al ✅ |
| `This iPhone cannot be used` | iTunes/Mobile Device driver niet OK | Herstart iPhone + Mac |
| `Vulkan SDK not found` | VULKAN_SDK niet gezet | `export VULKAN_SDK=~/VulkanSDK/macOS` |
| `MoltenVK.framework not found` | fetch-moltenvk.sh niet gedraaid | Herhaal stap 4 |
| App crashed direct | Missing dylib of framework | Check `ls Frameworks/` — 6 dylibs + MoltenVK.framework |
| Geen tekst/buttons in game | Fonts missen | `./scripts/build/ios/stage-fonts.sh` opnieuw |
| AltStore "No connected devices" | iPhone niet vertrouwd/gekoppeld | USB kabel, ontgrendeld, Trust this computer |

---

## CI-alternatief (geen Mac nodig)

Push naar `main` op GitHub → `.github/workflows/build-ios.yml` bouwt via GitHub Actions (macOS runner). Download `.ipa` artifact uit de Actions run.

```bash
# Alleen nodig als je assets wilt bundelen in CI:
gh workflow run "🍎 Build Generals Zero Hour — iOS" --ref main \
  -f skip_assets=false
```

Maar assets zijn 2.7GB en staan niet op de CI runner — die moet je lokaal toevoegen na download van het .ipa.

---

## Projectstructuur (snel overzicht)

```
Generals-iOS/
├── .github/workflows/build-ios.yml   # CI pipeline
├── scripts/build/ios/
│   ├── package-ios-zh.sh             # Signed packaging
│   ├── fetch-moltenvk.sh             # MoltenVK download
│   └── stage-fonts.sh                # Fonts voorbereiden
├── cmake/                            # CMake modules (openal, sdl3, gamespy, etc.)
├── ios/                              # Xcode project stub
├── CMakePresets.json                 # ios-vulkan preset
└── OPENCODE_BUILD.md                 # Dit bestand
```
