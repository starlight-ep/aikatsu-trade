# アンコール 交換申込

アイカツ！アンコールの交換・譲渡・買取の申込を受け付ける非公式Webアプリです。
カード帳（aikatsu-cardbook）の「譲」「求」をそのまま公開し、釣り合った内容でないと申込を送れないようにします。

## しくみ

| | 申し込む人 | ページの持ち主 |
| --- | --- | --- |
| URL | `…/aikatsu-trade/?u=ページID` | `…/aikatsu-trade/` |
| ログイン | 不要（裏で匿名ログイン） | カード帳と同じアカウント（Google／メール） |
| やること | 種類を選ぶ → カードを選ぶ → 申し込む → 交換コードの文章をXのリプに貼る | コードを照合 → 承認／お断り → 成立の文章をリプに貼る |

- 交換の種類：等価交換／比率交換（ポイント）／譲渡（価格）／買取（価格）
- ポイントや価格が釣り合わないと送信ボタンが押せない
- スパム対策：返事待ちは1端末1件まで、ブロック、受付停止、20枚まで、ボット用のダミー欄
- カード帳を使っている人なら誰でも、自分のページを作れる

## 公開手順

1. GitHub に `aikatsu-trade` リポジトリを作り、このフォルダの中身を push
2. Settings → Pages → Branch を `main` / `(root)` にして Save
3. Firebase コンソール（aikatsu-cardbook プロジェクト）で
   - Authentication → ログイン方法 →「匿名」を有効にする
   - Firestore → ルール を下の内容に置き換えて公開
4. `https://<ユーザー名>.github.io/aikatsu-trade/` を開いてログイン → ページを作る

※ カード帳とは別のGitHubアカウントに置いてもOKです（Firebaseの承認済みドメインに、そのアカウントの `<ユーザー名>.github.io` を追加してください）。

## Firestore のルール（カード帳の分も含めた全文）

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // カード帳：本人だけ
    match /users/{uid}/{document=**} {
      allow read, write: if request.auth != null && request.auth.uid == uid;
    }

    // 交換申込
    function profPath(h) { return /databases/$(database)/documents/trade_profiles/$(h); }
    function isOwner(h) { return request.auth != null && get(profPath(h)).data.uid == request.auth.uid; }

    match /trade_profiles/{h} {
      allow read: if true;
      allow create: if request.auth != null && request.auth.token.firebase.sign_in_provider != 'anonymous'
        && h.matches('^[a-z0-9_]{3,20}$') && request.resource.data.uid == request.auth.uid;
      allow update: if request.auth != null && resource.data.uid == request.auth.uid
        && request.resource.data.uid == resource.data.uid;
      allow delete: if request.auth != null && resource.data.uid == request.auth.uid;

      match /requests/{code} {
        allow read, update, delete: if isOwner(h);
        allow create: if request.auth != null
          && code.matches('^AE-[A-Z2-9]{6}$')
          && request.resource.data.keys().hasOnly(['code','uid','type','get','give','getPt','givePt','price','name','msg','status','createdAt'])
          && request.resource.data.code == code
          && request.resource.data.uid == request.auth.uid
          && request.resource.data.type in ['eq','ratio','sell','buy']
          && request.resource.data.get is list && request.resource.data.get.size() <= 20
          && request.resource.data.give is list && request.resource.data.give.size() <= 20
          && request.resource.data.name is string && request.resource.data.name.size() <= 30
          && request.resource.data.msg is string && request.resource.data.msg.size() <= 200
          && request.resource.data.status == 'pending'
          && request.resource.data.createdAt == request.time
          && get(profPath(h)).data.open == true
          && !exists(/databases/$(database)/documents/trade_profiles/$(h)/blocked/$(request.auth.uid))
          && existsAfter(/databases/$(database)/documents/trade_profiles/$(h)/devices/$(request.auth.uid));
      }

      // 返事待ちは1端末1件（承認／お断りで消える）
      match /devices/{u} {
        allow create: if request.auth != null && request.auth.uid == u
          && request.resource.data.keys().hasOnly(['code','createdAt'])
          && request.resource.data.createdAt == request.time
          && getAfter(/databases/$(database)/documents/trade_profiles/$(h)/requests/$(request.resource.data.code)).data.uid == u;
        allow read, delete: if isOwner(h);
      }

      match /history/{id} { allow read, write: if isOwner(h); }
      match /blocked/{u}  { allow read, write: if isOwner(h); }
    }
  }
}
```

## ファイル構成

| ファイル | 役割 |
| --- | --- |
| index.html | アプリ本体（申込画面・管理画面） |
| config.js | Firebase の設定（カード帳と同じ） |
| cards.js | カード一覧。新しいカードが出たら、カード帳の cards.js をコピーして差し替える |

## 注意

- 無断での複製・再配布・改変版の公開はご遠慮ください（LICENSE 参照）

- 公式のカード画像はリポジトリに含めず、公式サイトの画像を表示するだけです
- 本アプリはバンダイ公式とは関係ありません
