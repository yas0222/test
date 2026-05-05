# Android Capacitor 技術検証ガイド（にゃん・ノート）

このドキュメントは **Google Play 先行の技術検証** として、既存の Web/PWA 版を壊さずに Android 実機起動確認まで進めるための手順です。

> この時点では本番リリースは行いません。Play Console 登録・AAB アップロード・Production 申請は対象外です。

## 1. 前提条件

- Node.js（推奨: LTS 系）
- npm
- Android Studio（SDK / Emulator / Platform Tools を含む）
- Java / Gradle 環境
  - Android Studio 同梱 JDK を使う構成でも可

## 2. セットアップ手順（リポジトリ直下）

1. 依存関係をインストール

```bash
git pull origin main
```

```bash
npm install
```

```bash
rm -rf android
```

2. Capacitor 設定を確認

- `appId`: `app.nyannote.prototype`
- `appName`: `にゃん・ノート`
- `webDir`: `www`（Android 向けに静的ファイルをコピーして同期）

3. Android 用 Web アセットを生成

```bash
npm run prepare:android-web
```

4. Android プロジェクトを追加（初回のみ）

```bash
npm run cap:add:android
```

または次のコマンドでも実行できます。

```bash
npx cap add android
```

5. Web アセット同期

```bash
npm run cap:sync:android
```

または次のコマンドでも実行できます。

```bash
npx cap sync android
```

6. Android Studio で開く

```bash
npx cap open android
```

## 3. Android プロジェクト作成手順

1. 上記 `cap:add:android` 実行で `android/` が生成されます。
2. 必要に応じて `npm run cap:sync` で最新の Web 側変更を Android 側へ反映します。
3. Android Studio で開く

```bash
npm run cap:open:android
```

## 4. 実機またはエミュレーターでの起動手順

1. Android Studio で `android/` プロジェクトを開く
2. 実機を USB デバッグで接続、または Emulator を起動
3. Android Studio で Run 実行（または CLI から下記）

```bash
npm run android:run
```

4. アプリ起動後、主要画面が表示されることを確認
   - 一覧表示
   - 記録作成
   - 既存 PWA の基本導線が崩れていないこと

## 5. AAB 作成までの大まかな流れ（今回は未実施）

1. Android Studio で署名設定を準備
2. `Build > Generate Signed Bundle / APK` から AAB 生成
3. Play Console 内部テストへアップロード
4. テスト端末で起動・認証・保存動作を確認

> 今回は上記フローの「概念整理」のみで、アップロードや審査申請は行いません。

## 6. Google ログイン確認ポイント（要注意）

Capacitor（Android WebView）では、ブラウザ版と同じ挙動にならないケースがあります。

- `signInWithPopup` は WebView で失敗・制限されることがある
- 必要に応じて `signInWithRedirect` やネイティブ連携プラグインを検討
- Firebase Authentication で Android 向け設定（SHA-1 / SHA-256、OAuth クライアント）を再確認
- 認証後の戻り先とセッション維持を実機で確認

## 7. Firebase Authentication の追加確認点

- Authentication の承認済みドメイン
  - GitHub Pages / Firebase Hosting に加え、必要に応じて運用ドメインを確認
- Google プロバイダ有効化状態
- OAuth 同意画面・クライアント設定
  - Android パッケージ名と署名フィンガープリント対応
- リダイレクトフロー利用時のハンドラ到達性

## 8. 既存 PWA 版を壊さないための注意点

- `index.html` / `app.js` / `manifest.json` / `sw.js` の既存仕様を変更しない
- Firestore Rules・Firebase 設定・Firestore データには手を加えない
- Capacitor は追加レイヤーとして扱い、Web 公開導線（GitHub Pages）は維持
- Web 側を更新したら `npm run cap:sync:android` のみ追加で実施

## 9. 技術検証の完了基準

- 既存 Web/PWA 版が従来どおり表示・操作できる
- Capacitor 最小構成（`package.json` / `capacitor.config.ts`）がある
- Android Studio で開いて Run できる準備が整っている
- 本ドキュメントに手順と注意点がまとまっている

## 10. Android WebView で Google 認証が不安定な場合の移行方針

`signInWithPopup` / `signInWithRedirect` を実装しても、端末固有の WebView 制限や Cookie 制約で復帰に失敗する場合は、`@capacitor-firebase/authentication` などのネイティブ認証方式へ段階的に移行する。

推奨ステップ:

1. **適用範囲を Android/Capacitor のみに限定**
   - Web/PWA は既存 Firebase Web Auth（Popup/Redirect）を維持する。
   - `isCapacitorNativePlatform()` の判定で分岐し、既存ブラウザ動線を壊さない。

2. **ネイティブ Google サインインを取得**
   - `@capacitor-firebase/authentication` の `signInWithGoogle()` で ID トークンを取得する。
   - Android 側は SHA-1 / SHA-256、`google-services.json`、OAuth クライアントを Firebase Console で整合させる。

3. **Firebase Auth へ credential 連携**
   - 取得した ID トークンから `GoogleAuthProvider.credential()` を作成し、`signInWithCredential`（または匿名 UID への link）を実行する。
   - 匿名ユーザーからの昇格失敗時はローカルデータ保持を優先し、移行案内を UI 表示する。

4. **復帰失敗時のフォールバック UI を固定**
   - 認証失敗時でもホーム画面表示を維持し、エラーコード・次アクション（再試行/後で実施）を明示する。
   - `?authDebug=1` で Web/Native どちらのフローを通ったか判別可能にする。

5. **段階導入**
   - まず内部テストで Android のみ native auth を有効化。
   - 問題なければ正式リリースへ展開し、Web/PWA ルートは変更しない。
