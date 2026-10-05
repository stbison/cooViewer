# Apple Silicon (arm64) ネイティブ化メモ

2026-09-21 / Xcode 27.0 (27A266a) / macOS 26 で確認。

## 変更点

- 同梱していた `XADMaster.framework` / `UniversalDetector.framework`（x86_64 専用バイナリ）を削除
- 上流 [MacPaw/XADMaster](https://github.com/MacPaw/XADMaster) (137728c) と
  [MacPaw/universal-detector](https://github.com/MacPaw/universal-detector) (4eb832d) から
  静的ライブラリ `libXADMaster.a` / `libUniversalDetector.a` を Universal (arm64 + x86_64) でビルドし、`Libraries/` に同梱
  （上流には framework ターゲットがもう無く、静的ライブラリのみ）
- `project.pbxproj`:
  - `ARCHS = "arm64 x86_64"`、`MACOSX_DEPLOYMENT_TARGET = 12.0`（Xcode 27 の下限）
  - `HEADER_SEARCH_PATHS = $(SRCROOT)/Libraries/include`、`LIBRARY_SEARCH_PATHS = $(SRCROOT)/Libraries`
  - `OTHER_LDFLAGS = -ObjC -all_load -lXADMaster -lUniversalDetector -lz -lbz2 -lc++`
    - **`-ObjC -all_load` が無いと、zip/rar 等のパーサクラスがリンカに捨てられて
      「broken or not image file」になる**（静的ライブラリ化に伴う唯一の罠）
- ソースコードの変更（Apple Silicon 化そのものには不要だが、フルスクリーン運用で実害があったもの）:
  - `CustomWindow.m`: タイトルバー高さを 22px 決め打ち → 実測値（PR #27 kanjitalk755 氏の修正を取り込み。macOS 26 では 32px で、旧コードは下 10px が切れる）
  - `Controller.m` / `CustomImageView.m`: ページバー（AccessoryWindow、親の子ウィンドウ）が macOS の親ウィンドウ自動移動で**位置を取り残される**（実測で再現） → 親の move / resize / becomeKey / changeScreen で再配置。併せて、`hidesOnDeactivate` で親が隠れて戻った時に子が再表示されない／親の下に潜る経路への備えとして、見えていなければ親の直上に出す（この経路は未観測の予防措置）
    - ⚠️ **「ページバーが消える」は本修正とは別件だった**（2026-10-05 実測）。真因は `o` キー（本家の既定割当・修飾なし）が「ページバー表示」のトグルで、押すと `defaults` の `ShowPageBar` が 0 になり保存されること。復帰は `o` をもう一度押す
  - `CustomWindow.m`: 解像度・ディスプレイ構成の変更で旧サイズのまま残る → `NSApplicationDidChangeScreenParametersNotification` でフルスクリーン矩形を再計算。**この通知はメニューバーの表示/非表示でも飛ぶ**ので、画面の実寸（`frame`）が変わった時だけ処理する（ガード無しだと、メニューバーが出た瞬間に消すループになる）

## ビルド

```bash
xcodebuild -project cooViewer.xcodeproj -target cooViewer -configuration Deployment CODE_SIGN_IDENTITY="-" build
```

## 静的ライブラリの再生成（上流を更新したい時）

```bash
git clone https://github.com/MacPaw/XADMaster.git
git clone https://github.com/MacPaw/universal-detector.git UniversalDetector   # 隣に置く（XADMaster が相対参照）
xcodebuild -project UniversalDetector/UniversalDetector.xcodeproj -target libUniversalDetector.a -configuration Release ARCHS="arm64 x86_64" ONLY_ACTIVE_ARCH=NO MACOSX_DEPLOYMENT_TARGET=12.0 SYMROOT=$PWD/build build
xcodebuild -project XADMaster/XADMaster.xcodeproj -target libXADMaster.a -configuration Release ARCHS="arm64 x86_64" ONLY_ACTIVE_ARCH=NO MACOSX_DEPLOYMENT_TARGET=12.0 SYMROOT=$PWD/build build
cp build/Release/lib*.a cooViewer/Libraries/
cp XADMaster/*.h cooViewer/Libraries/include/XADMaster/
cp UniversalDetector/UniversalDetector.h cooViewer/Libraries/include/UniversalDetector/
```

## 検証済み

- `lsappinfo` で `LSArchitecture = arm64` を確認
- フォルダ直開き / UTF-8 名 cbz / Shift_JIS(cp932) 名 cbz の 3 種で表示・ページ送り OK
- Shift_JIS 名の自動判定は旧同梱版（"3.10 libxad 13.0, modified"）と同一結果（Rosetta で並走比較）
