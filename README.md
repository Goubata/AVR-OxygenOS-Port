# OxygenOS 17 for Motorola AVR（非公式移植）

OnePlus 15 の **OxygenOS 17 GLO（CPH2747_17.0.0.102）** を、Motorola **AVR（XT2611、Snapdragon 8 Gen 5 "canoe"）** に移植した非公式 ROM です。このリポジトリはリリース配布専用です（IMG は [Releases](../../releases) にあります）。

> **注意**：非公式のビルドです。自己責任で使ってください。焼く前に必ずバックアップを取ってください。中のファイルの権利は OnePlus／Motorola などの各社にあります。

## 配布物（Releases）
| ファイル | 用途 |
|---|---|
| `AVR-OxygenOS17-vX-system.img.zst.partNN` | COS の system（system_ext・product・my_* を同梱） |
| `AVR-OxygenOS17-vX-odm.img.zst.partNN` | 移植版の odm（AVR ストックの odm＋カメラ・ゲーム機能用のファイル）。**v1.2 は odm に変更がないため置いていません。v1.1 の odm を使ってください** |
| `SHA256SUMS.txt` | 展開後の IMG と分割ファイルのハッシュ |

結合と展開（Linux／Termux／macOS）：
```sh
cat AVR-OxygenOS17-vX-system.img.zst.part* > system.img.zst && zstd -d system.img.zst
cat AVR-OxygenOS17-vX-odm.img.zst.part*    > odm.img.zst    && zstd -d odm.img.zst
sha256sum -c SHA256SUMS.txt --ignore-missing
```
Windows では `copy /b` で結合し、zstd で展開してください。

## v1.1 から v1.2 への更新（データはそのまま）
v1.2 で変わったのは system だけです。fastbootd で system だけを焼き直します（`-w` は付けません）。
```sh
fastboot reboot fastboot
fastboot flash system system.img
fastboot reboot
```
- odm は v1.1 のままで構いません。
- 念のため、更新前にバックアップを取ってください。

## 焼き方（fastbootd）
前提：ブートローダーのアンロック、AVR（XT2611）、スロット a。**データは全部消えます。**
```sh
# 1. vbmeta の検証を無効にする（まだの場合。自分の端末の vbmeta.img を使う）
fastboot --disable-verity --disable-verification flash vbmeta_a vbmeta.img
fastboot set_active a
# 2. fastbootd で、使わない product / system_ext を消して空きを作り、system と odm を焼く
fastboot reboot fastboot
fastboot delete-logical-partition product_a
fastboot delete-logical-partition system_ext_a
fastboot delete-logical-partition system_b
fastboot flash system system.img
fastboot flash odm odm.img
fastboot -w
fastboot reboot
```
- product と system_ext は COS の system に同梱されていて、COS の設定（skip_mount.cfg）でマウントされないので不要です。
- `system_b` が存在しない場合のエラーは無視して構いません。
- DSU（動的システム更新）で試す場合は、system.img と odm.img を DSU ツールで導入してください。

元に戻す：ストックのファームウェアを焼き直してください。

## 動作状況（開発者の実機で確認。v1.1 以降は実機に焼いた状態）
**動作するもの**：起動・操作、電源キー、振動（強弱・COS の振動パターン）、指紋（光る・登録・認証・**画面オフ解除**＝設定「画面オフでも指紋アイコンを表示する」）、Motorola カメラ（全機能）、顔認証、Wi-Fi、SMS 受信、VoLTE、COS の音響モード、色モード（標準・自然）、明るさ・色温度、144 Hz、ステータスバー、ダブルタップで点灯、持ち上げて点灯、タッチブースト、ゲームアシスタントのバイパス充電、急速充電の表示（90 W）、サーマル制御（Motorola の thermal-engine、ストックと同じ温度センサー）

**v1.1 の実機で確認**：ゲームアシスタントの**フレーム補間・超解像**、実機に焼いた状態での起動

**実装済み・未確認**：**v1.2 の 3 件**（Bluetooth の音量、パンチホール位置、充電制限→バイパス給電）、見ている間は画面を消さない（Google Android System Intelligence の通常カメラ方式）、ゲームの画質調整（Qualcomm の Adreno プロファイル経由）、USB の水濡れ警告

**未対応・既知の問題**：HDR 動画で画面が HDR の明るさにならない（YouTube が HDR 映像を自前で SDR に変換して表示しているため。端末側の HDR 対応の申告は正常）、色モードの「鮮明」、画面オフの手書きジェスチャー（O・V・M・W）、低電力カメラ（AON）を使う機能、COS カメラは初期状態で無効（Motorola カメラを使用）

## 変更履歴
- **v1.2**（2026-11-03）：system のみ更新（odm は v1.1 のまま）。
  - **Bluetooth の音量が効かない**問題を修正。OnePlus の Bluetooth が一部のイヤホン（例：soundcore Liberty 5）をカーナビと誤判定して、機器側の音量連動（AVRCP Absolute Volume）を切っていました。誤判定の機能を止め、普通の Android と同じ動作にしました。
  - **パンチホール**：ステータスバーなどが、カメラ穴に対して少し（9 px）上にずれていたのを、ストックの穴の位置に合わせました。
  - **カスタム充電制限**（バッテリー設定）：指定した % に達すると、ゲームアシスタントのバイパス給電と同じ経路に切り替えて充電を止めます。指定値から 2% 下がると充電を再開します。ゲームアシスタントのバイパス給電とは別々に動きます。
  - いずれも実機では未確認です。
- **v1.1**（2026-11-02）：ゲームアシスタントのフレーム補間・超解像を移植。
  - 超解像は COS の Colorx（GPU シェーダー）。フレーム補間は Colorx（GPU）と OnePlus の gameinference サービス（AI 補間を NPU で実行）。
  - AVR のカーネルに無い `/dev/oplus_sync`（ソフトウェア同期フェンス）を、モジュール `avr_oplus_sync.ko`（odm に同梱、起動時に自動で読み込み）で追加。
  - 機能の中継役の oiface を再び有効化。ゲーム中に CPU の周波数の下限を引き上げることがあります。発熱による上限の制御（Motorola の thermal-engine）はそのまま有効です。
  - 実機では未確認です。
- **v1.0**（2026-11-02）：初回リリース。開発用の診断ログ（起動後 1 時間の記録、スロットリング・カメラなどの診断）を削除。DSU 専用だった判定を外し、焼いた状態でも全修正が有効になるようにした。

---

## English (short)
Unofficial port of **OxygenOS 17 GLO (OnePlus 15, CPH2747_17.0.0.102)** to the **Motorola AVR (XT2611, SD 8 Gen 5)**. Release-only repository; images (system, odm) are in Releases. **v1.2** (system only; keep the v1.1 odm): Bluetooth volume fix (OPlus carkit misdetection disabled absolute volume), punch-hole UI position, custom charging limit uses bypass charging. Update from v1.1 without wiping: in fastbootd, `fastboot flash system system.img`.
Flash (unlocked bootloader, slot a, wipes data): disable vbmeta verification, then in fastbootd delete `product_a`, `system_ext_a`, `system_b`, flash `system.img` and `odm.img`, `fastboot -w`, reboot. Join the parts with `cat`, decompress with `zstd -d`, check with `SHA256SUMS.txt`. Use at your own risk; tested by the developer via DSU only.
