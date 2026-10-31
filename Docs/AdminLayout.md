# 管理者権限と固定サイズ床

以下の床の手順は protocol 19 の実装です。実機での位置合わせ精度は設置位置と向きに依存します。

## 署名された管理者

既存の Ed25519 / EdDSA JWT に、厳密な `role="admin"` を追加します。9 claim、署名付き DTLS fingerprint、発行者・audience・有効期限・single-use jti の要件は同じです。admin を含め、サーバー・クライアントには発行者の秘密鍵を渡しません。再接続には新しいトークンが必要です。

管理者は専用の観戦者です。fighter slot は常に -1、射撃・シールド・プレイヤーの動画／実測形状の発行権限はありません。サーバーで検証した signed role だけから `bAdmin` を設定します。`?Admin=1`、UI 操作、ローカル PlayerState の書換えでは player / inspector トークンを昇格できません。すべての管理 RPC は現在の認証状態・期限・role・revision・rate をサーバーで再検査します。

プレイヤーは2人、管理者は1人までです。観戦者はプレイヤー枠・管理者枠とは別に数えます。

```powershell
# 発行は既存の信頼された発行者 PC で行う。実際の秘密値をコマンドへ直書きしない。
.local/auth-tools/Scripts/python.exe tools/auth/mrfps_tokens.py issue `
  --private-key <issuer-private.pem> --passphrase-file <protected-passphrase-file> `
  --subject operator-1 --role admin --dtls-fingerprint <current-server-fingerprint> `
  --output <new-operator.jwt>

# 全ターゲットを同じ protocol 19 へそろえてから接続する。
UnrealEditor.exe MRFPS.uproject /Game/Maps/MRFPSArena -game `
  -MRFPSServer=<server:port> -MRFPSAdmin -MRFPSDesktop -nohmd `
  -MRFPSAuthPublicKeyFile=<issuer-public.pem> -MRFPSAuthTokenFile=<new-operator.jwt>
```

F1 の接続メニューに Join admin、接続後 F2 に管理パネルを追加します。Start / Pause / Resume / Reset、床の登録／CALIBRATION、寸法設定、壁配置／消去を行えます。一般の観戦者に管理パネルや昇格ボタンは提供しません。

## 固定寸法と床の校正

初期値は **X 幅 450 cm、Y 長さ 700 cm**。INI の `ArenaWidthCm` / `ArenaLengthCm` または管理者の明示的な寸法操作でのみ変えます。範囲は各 200〜2000 cm です。寸法は実測された二人の距離へ合わせません。追跡のスケールも変更しません。

両プレイヤーは実際の開始位置に立ち、互いに向き合ってそれぞれ左 Y を押します。端末ごとに、その位置と向きを固定床の P1/P2 の両端と内向きの方向へ対応させます。両端の実距離は設定した床の長さに合わせ、両端の床の高さもそろえてください。これは手動の位置合わせであり、実際の距離や向きの誤差を自動測定するものではありません。その後、管理者が「原点登録」を押します。

登録条件は、有効な署名 player の 0/1 slot、双方の新鮮で妥当な追跡、校正済み、Stage floor、非 desktop、現在の Pawn identity と校正世代の一致です。Stage 床はその共通基準の Z=0 です。頭の Z の平均から床高を作りません。

**CALIBRATION を押した時だけ**、二人の現在の頭位置の XY 中点を床中心、player0→player1 の平面方向を床 +Y にします。二人の実測位置は動かしません。床のサイズは 450×700 cm のままです。たとえば実測距離が 6 m でも床は 7 m です。実測距離は独立した値として UI に表示します。

歩行は床を追従・再中心化させません。明示操作の間、床の中心・向き・サイズは静止します。再校正・recenter・追跡喪失・認証失効・プレイヤー交代で登録が無効になったら、最後の床変換を保持して `UNAVAILABLE` とし、共有物理形状と戦闘を止めます。再開には再登録／CALIBRATION／明示 Resume が必要です。epoch のみが進み、勝手な再計算は行いません。

desktop の初期位置／再出現位置は、床ローカルの `(0,-Length/2,170)` と `(0,+Length/2,170)`、互いを向く方向です。これは **nominal desktop input** で、実際の人体位置とは表示しません。XR の位置や追跡変換をこの値へテレポートさせません。二人とも desktop の開始は `DESKTOP NOMINAL` と表示し、物理床校正と区別します。desktop と XR が混在する試合は共通床の物理校正済みとして開始できません。

## 戦闘と壁

青／橙の床マーカーは設定された Y 両端の **nominal start** を示します。実測された人体位置として発行するものではありません。壁配置中は動画パネルを一時的に隠し、配置を終了したら元のパネル表示へ戻します。動画上の画素を床座標へ転用しません。

既定で `bRequireAdminStart=True`。二人が用意できても自動開始しません。管理者が Start します。試合終了後は明示 Reset で待機へ戻します。サーバー所有の設定で `False` にする既存の自動試験／nominal desktop 運用は残します。

Pause は UE の world pause を使いません。射撃・ダメージ・シールド消費／回復・壁ジェスチャー・再出現・クールダウン・fighter 壁の寿命だけを止めます。追跡、実映像、実測形状の鮮度、ネットワーク、認証期限、UI は進み続けます。Resume では残りの戦闘時間を復元します。Pause / Resume / Reset は firing を解除し、MoBits を STOP します。再 ARM は操作者の明示操作です。

PLACE WALLS は自由な 3D 観戦ビューへ切り替えてカーソルを解放します。viewport の床をクリックすると、サーバーが ray と現在の固定床面の交点を計算し、壁全体の四隅・占有・layout revision・操作頻度・上限を検査して実際の replicated barrier を作ります。動画面のクリックを座標測定として扱いません。メニューのクリックでは配置／再キャプチャしません。Esc で配置を取り消し、カーソルを解放したままにします。従来の観戦者の Esc 解放／viewport click 捕捉、プレイヤーの Escape STOP を保持します。

管理壁は厚さ 16、幅 160、高さ 200 cm、所有 fighter なし。弾は遮りますが銃では壊れず、最大 32 個、4 回/秒以下で設置でき、削除または Reset / Clear まで残ります。通常 fighter 壁も銃では壊れませんが、6 秒寿命・4 秒 cooldown は引き続き有効です。床の明示的な変更では既存壁と実測形状を消し、古い世界座標の物体を新しい床へ流用しません。

## 形状プロトコルと適用

`FMRFPSGeometryChunk` に `LayoutRevision`、stop RPC に同 revision を追加します。各 fragment は allocation 前に count/length を検査し、追加後も 1024-byte native DTLS に収まる 600 byte 未満です。wire の座標 envelope と、現在の回転した固定床への包含判定を分けます。source / server / renderer commit が現在の layout revision と bounds を照合します。layout 更新は geometry assembler・queue・PS stream・CPU collision・mesh を無効化し、旧 layout の frame / stop で新世界の物体を閉じたり表示したりできないようにします。

**network protocol 19** が必要です。Editor / Windows / Linux / Quest を同じ版へ更新し、稼働サーバーの新しいDTLS fingerprintに署名された各roleのトークンを発行します。映像の30fps目標、rate admission、受信公平性、captureの停止・復帰、ネットワーク速度の変更と統合済みです。30fpsの実機達成を示すものではありません。

## 実機での制約

対向した2台のQuestによる校正と、今回のサーバー側IK当たり判定は、実機での動作を確認していません。
