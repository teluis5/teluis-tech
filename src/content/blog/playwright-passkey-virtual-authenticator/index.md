---
title: 'Playwrightでパスキー（WebAuthn）認証を自動テストする！仮想オーセンティケーター完全攻略'
description: 'PlaywrightとChrome DevTools Protocol (CDP) の仮想オーセンティケーター（Virtual Authenticator）を使い、パスキー（WebAuthn/FIDO2）の登録・ログインフローをE2Eテストで自動化する方法を徹底解説します。'
pubDate: '2026-09-09'
categories: ['Playwright', 'テスト自動化', 'WebAuthn', 'セキュリティ']
---

パスキー（WebAuthn / FIDO2）の普及に伴い、新規登録やログインの主要手段としてパスキーを採用するWebサービスが急速に増えています。

しかし、QAやテスト自動化の現場で必ずぶつかるのが**「生体認証（Touch ID、Windows Hello）やセキュリティキーのタッチをどうやって自動化するか？」**という問題です。当然ながら、GitHub Actions等のCI/CD環境やヘッドレスブラウザでは物理的な指紋認証や顔認証は行えません。

そこで本記事では、Playwrightからブラウザ内部の**仮想オーセンティケーター（Virtual Authenticator）**を制御し、パスキー認証の登録・ログインを自動テストする実践的な方法を詳しく解説します。

---

## 仮想オーセンティケーター（Virtual Authenticator）の仕組み

WebAuthn API（`navigator.credentials.create()` や `navigator.credentials.get()`）がブラウザで実行されると、通常はOSのダイアログが起動し、ユーザーに生体認証やセキュリティキーのタッチを求めます。

Chromiumには**仮想認証環境（Virtual Authenticator Environment）**が備わっており、これを有効化すると、ブラウザ内部のソフトウェア認証器がWebAuthnの呼び出しをインターセプトします。物理デバイスを介さず、あらかじめ指定したパラメータに従って即座に公開鍵や署名を返却してくれる仕組みです。

Playwrightでは **CDP（Chrome DevTools Protocol）セッション** を介してこの仮想認証環境にアクセスし、自由に認証器の作成・操作・検証が行えます。

---

## 1. 仮想認証器のセットアップとオプション解説

まずはPlaywrightのCDPセッションを使って仮想オーセンティケーターを作成する基本形です。

```typescript
import { test, expect } from '@playwright/test';

test('仮想認証器をセットアップする', async ({ page }) => {
  // 1. CDPセッションを開始
  const cdp = await page.context().newCDPSession(page);

  // 2. WebAuthn環境を有効化
  await cdp.send('WebAuthn.enable');

  // 3. 仮想オーセンティケーターを追加
  const { authenticatorId } = await cdp.send('WebAuthn.addVirtualAuthenticator', {
    options: {
      protocol: 'ctap2',                 // FIDO2 / CTAP2 を指定
      transport: 'internal',             // internal: スマホ・PC内蔵のプラットフォーム認証器
      hasResidentKey: true,              // パスキー（Discoverable Credential）に必須
      hasUserVerification: true,         // 生体認証やPINによるユーザー検証に対応
      isUserVerified: true,              // ユーザー検証を自動的に「成功」としてシミュレート
      automaticPresenceSimulation: true, // 物理タッチ・存在確認を自動成功
    },
  });

  console.log(`作成された認証器ID: ${authenticatorId}`);
});
```

### パラメータのポイント

| オプション名 | 推奨値 | 解説 |
| :--- | :--- | :--- |
| `protocol` | `'ctap2'` | パスキー（FIDO2）をテストする場合は必ず `'ctap2'` を指定します。 |
| `transport` | `'internal'` | Touch ID や Windows Hello などのプラットフォーム認証器を模倣します。外付けUSBキーを模倣したい場合は `'usb'` を指定します。 |
| `hasResidentKey` | `true` | **パスキーにおいて最も重要な設定**です。認証器自体にクレデンシャル情報（秘密鍵・ユーザー識別子）を保持する「Discoverable Credentials」を有効化します。 |
| `hasUserVerification` | `true` | 認証器が指紋・顔・PINなどの本人確認能力を持っていることを示します。 |
| `isUserVerified` | `true` | リクエスト時のユーザー検証（UV）を成功させます。`false` にすることで生体認証失敗テストが可能です。 |
| `automaticPresenceSimulation`| `true` | 物理タッチ待機をスキップし、即座に応答を返します。自動テストでは必須です。 |

---

## 2. 実践：パスキー登録＆ログインのテストシナリオ

Webアプリケーションにおける「パスキー登録」と「パスキーログイン」の一連のフローをテストするコード例です。

```typescript
import { test, expect } from '@playwright/test';

test.describe('パスキー認証E2Eテスト', () => {
  let cdp: any;
  let authenticatorId: string;

  test.beforeEach(async ({ page }) => {
    // テスト開始前に仮想認証器をセットアップ
    cdp = await page.context().newCDPSession(page);
    await cdp.send('WebAuthn.enable');

    const result = await cdp.send('WebAuthn.addVirtualAuthenticator', {
      options: {
        protocol: 'ctap2',
        transport: 'internal',
        hasResidentKey: true,
        hasUserVerification: true,
        isUserVerified: true,
        automaticPresenceSimulation: true,
      },
    });
    authenticatorId = result.authenticatorId;
  });

  test.afterEach(async () => {
    // 終了後に認証器を破棄
    if (cdp && authenticatorId) {
      await cdp.send('WebAuthn.removeVirtualAuthenticator', { authenticatorId });
      await cdp.send('WebAuthn.disable');
    }
  });

  test('パスキーの新規登録からログインまで完走できること', async ({ page }) => {
    // --- 1. アカウント設定画面でパスキーを登録 ---
    await page.goto('https://example.com/settings/security');
    
    // 「パスキーを登録」ボタンをクリック
    // ブラウザ内で navigator.credentials.create() が呼ばれ、仮想認証器が自動応答する
    await page.getByRole('button', { name: 'パスキーを登録する' }).click();

    // 登録成功トースト・メッセージの確認
    await expect(page.getByText('パスキーが正常に登録されました')).toBeVisible();

    // --- 2. 仮想認証器内にクレデンシャルが保存されたか検証 ---
    const { credentials } = await cdp.send('WebAuthn.getCredentials', { authenticatorId });
    expect(credentials.length).toBeGreaterThan(0);

    // --- 3. ログアウト ---
    await page.getByRole('button', { name: 'ログアウト' }).click();

    // --- 4. パスキーでログイン ---
    await page.goto('https://example.com/login');

    // 「パスキーでログイン」をクリック
    // navigator.credentials.get() が呼ばれ、仮想認証器が署名を返却する
    await page.getByRole('button', { name: 'パスキーでログイン' }).click();

    // ログイン成功後のダッシュボード遷移を確認
    await expect(page).toHaveURL(/.*dashboard/);
    await expect(page.getByRole('heading', { name: 'ダッシュボード' })).toBeVisible();
  });
});
```

---

## 3. カスタムFixturesでテストコードを共通化する

テストファイルごとにCDPセッションの初期化や後始末を書くのは冗長です。Playwrightの `test.extend` を使ってカスタムFixtureを作ると、宣言的で非常にメンテナンス性の高いコードになります。

### `fixtures/webauthn.ts`

```typescript
import { test as base, type CDPSession } from '@playwright/test';

type WebAuthnFixture = {
  virtualAuth: {
    cdp: CDPSession;
    authenticatorId: string;
    getCredentials: () => Promise<any[]>;
    clearCredentials: () => Promise<void>;
    setUserVerified: (verified: boolean) => Promise<void>;
  };
};

export const test = base.extend<WebAuthnFixture>({
  virtualAuth: async ({ page }, use) => {
    const cdp = await page.context().newCDPSession(page);
    await cdp.send('WebAuthn.enable');

    const { authenticatorId } = await cdp.send('WebAuthn.addVirtualAuthenticator', {
      options: {
        protocol: 'ctap2',
        transport: 'internal',
        hasResidentKey: true,
        hasUserVerification: true,
        isUserVerified: true,
        automaticPresenceSimulation: true,
      },
    });

    // テスト本体に渡すヘルパー
    await use({
      cdp,
      authenticatorId,
      getCredentials: async () => {
        const res = await cdp.send('WebAuthn.getCredentials', { authenticatorId });
        return res.credentials;
      },
      clearCredentials: async () => {
        await cdp.send('WebAuthn.clearCredentials', { authenticatorId });
      },
      setUserVerified: async (verified: boolean) => {
        await cdp.send('WebAuthn.setUserVerified', {
          authenticatorId,
          isUserVerified: verified,
        });
      },
    });

    // テスト終了後のクリーンアップ
    await cdp.send('WebAuthn.removeVirtualAuthenticator', { authenticatorId });
    await cdp.send('WebAuthn.disable');
  },
});

export { expect } from '@playwright/test';
```

### Fixtureを使ったテストの記述

```typescript
import { test, expect } from './fixtures/webauthn';

test('パスキーでスムーズにログインできること', async ({ page, virtualAuth }) => {
  await page.goto('https://example.com/login');
  
  // 仮想認証器が有効化されているため、クリックするだけで認証完了
  await page.getByRole('button', { name: 'パスキーでログイン' }).click();

  await expect(page).toHaveURL(/.*dashboard/);
});
```

---

## 4. エッジケース・異常系のテストパターン

仮想認証器の最大の利点は、**「実機では再現が面倒な異常系・エラーパターン」をプログラムから自在にテストできる点**にあります。

### パターン①：生体認証（ユーザー検証）に失敗したケース

ユーザーが指紋認証に失敗したときのUI挙動を検証したい場合、`setUserVerified(false)` を設定します。

```typescript
test('ユーザー検証失敗時に適切なエラーが表示されること', async ({ page, virtualAuth }) => {
  // ユーザー検証を「失敗」に切り替え
  await virtualAuth.setUserVerified(false);

  await page.goto('https://example.com/login');
  await page.getByRole('button', { name: 'パスキーでログイン' }).click();

  // アプリケーション側で適切なエラーハンドリングが行われているか検証
  await expect(page.getByText('認証に失敗しました。もう一度お試しください。')).toBeVisible();
});
```

### パターン②：該当するパスキーが存在しないケース

端末にパスキーが登録されていない状態でログインを試みた場合：

```typescript
test('パスキー未登録時に適切なメッセージが表示されること', async ({ page, virtualAuth }) => {
  // 認証器内のクレデンシャルを空にする
  await virtualAuth.clearCredentials();

  await page.goto('https://example.com/login');
  await page.getByRole('button', { name: 'パスキーでログイン' }).click();

  await expect(page.getByText('登録されたパスキーが見つかりませんでした')).toBeVisible();
});
```

---

## 5. 導入時の注意点・ハマりどころ

1. **Chromium 限定であること**
   - CDPベースの仮想オーセンティケーター機能は、Chromium（Chrome / Edge）でのみ利用可能です。FirefoxやWebKit（Safariエンジン）のプロジェクトでは動作しないため、`playwright.config.ts` でWebAuthnテスト用のプロジェクトをChromium系に限定する構成にしておきましょう。
2. **セキュアコンテキスト（HTTPS / localhost）の必須**
   - WebAuthnはセキュリティ上の理由から、セキュアコンテキスト（`https://` または `http://localhost`）でのみ動作します。ローカル開発環境やCIで実行する際は、ホスト名が要件を満たしているか確認してください。
3. **RP ID（Relying Party ID）の整合性**
   - アプリケーションが指定する `rp.id`（例: `example.com`）とテスト対象のドメインが一致していないと、ブラウザによって `SecurityError` が発生します。ステージングやプレビュー環境でドメインが動的に変わる場合は注意が必要です。

---

## まとめ

PlaywrightとCDPの仮想オーセンティケーターを組み合わせることで、これまで手動テストに頼りがちだったパスキー認証の自動テストが簡単に実現できます。

- **`WebAuthn.addVirtualAuthenticator`** でプラットフォーム認証器をブラウザ内に構築
- **`hasResidentKey: true`** でパスキー（Discoverable Credentials）を完全シミュレート
- **カスタムFixtures** を用意すればテストコードをシンプル・共通化可能
- **`setUserVerified`** 等で生体認証失敗などの異常系も網羅

パスキー導入時のリグレッション防止や、日々のデプロイ前のCIパイプラインにぜひ取り入れてみてください。
