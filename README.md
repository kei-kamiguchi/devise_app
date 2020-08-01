# devise
## 導入
1. gem`devise`をインストール
2. deviseの設定ファイルをrailsアプリケーションにインストール
```
$ rails g devise:install
```
3. `root_path`を設定
4. フラッシュメッセージの設定
```
<% if notice %>
  <p class="alert alert-notice"><%= notice %></p>
<% end %>
<% if alert %>
  <p class="alert alert-error"><%= alert %></p>
<% end %>
```
5. deviseに関する一通りのviewsファイルを自動的に作成
```
rails g devise:views
```
6. Userモデルを作成
```
$ rails generate devise user
```
7. マイグレーションを実行
## ヘルパーメソッド
```
# ログイン済のユーザーだけにアクセスを許可する場合に使用
before_action :authenticate_user！
# ユーザーのログイン有無をboolean値で返すメソッド
user_signed_in？
# 現在ログインしているユーザの情報を返すメソッド
current_user
# 現在ログインしているユーザのセッション情報を返すメソッド
user_session
```
8. リンクの設定
```
<%= link_to "ログアウト", destroy_user_session_path, method: :delete %>
<%= link_to "ログイン", new_user_session_path %>
```
## deviseのモジュール
### デフォルト
```
database_authenticatable
# ログイン時のパスワードを暗号化してデータベースに保存する機能
registerable
# ユーザーの登録・編集・削除処理を実行する機能
recoverable
# パスワードリセット機能
rememberable
# cookieにユーザー情報を持たせる
trackable
# サインインした回数や時刻、IPアドレスを記録
validatable
# メールアドレスとパスワードに関するデフォルトのバリデーションを提供
```
### 追加モジュール
```
lockable
# サインインに一定回数失敗するとアカウントをロックさせる機能
confirmable
# ユーザー登録時に確認メールを送り、その中にあるURLをクリックすることで正式登録とする機能
timeoutable
# 一定時間使われていないセッション情報を削除する機能
omniauthable
# OmniAuthサポートする(SNSログインなどに必須)
```
## カラムの追加
[app/controllers/application_controller.rb]
```
before_action :configure_permitted_parameters, if: :devise_controller?

protected

# ユーザー登録時に「name」カラムを追加
def configure_permitted_parameters
  devise_parameter_sanitizer.permit(:sign_up, keys: [:name])
end

# ユーザー情報更新時に「name」カラムを追加
def configure_permitted_parameters
  devise_parameter_sanitizer.permit(:account_update, keys: [:name])
end
```
## 日本語化
1. デフォルト言語を日本語に設定
```
~(省略)~
module DeviseTest
  class Application < Rails::Application
    ~(省略)~
    config.load_defaults 5.1
    config.i18n.default_locale = :ja #これを追記
    ~(省略)~
  end
end
```
2. gem`devise-i18n`をインストール
3. devise-i18nのgem本体に存在するビューファイルをすべて自分のプロジェクト内にコピー(辞書ファイルは devise-i18n gem内部にある)
```
$ rails g devise:i18n:views
# 上書きするか？と聞かれているので、キーボードの「y」（yes の意）を押す。これを、同様のメッセージが表示されなくなるまで繰り返す
```
4. `devise-i18n`の辞書ファイルを手元で内容を確認したい場合や変更を加えたい場合、以下のコマンドを実行
```
$ rails g devise:i18n:locale ja
```
## コントローラーのカスタマイズ
1. deviseのコントローラーをプロジェクト内に生成
```
$ rails generate devise:controllers users
```
## メールアドレス認証による新規登録の実装
1. [model/user.rb]に`confirmable`モジュールを追加
2. Confirmableの機能を使うのに必要なカラムの追加
```
$ rails g migration add_confirmable_to_devise
```
[db/migrate/xxxxxxxxx_add_confirmable_to_devise.rb]
```
def up
  add_column :users, :confirmation_token, :string
  add_column :users, :confirmed_at, :datetime
  add_column :users, :confirmation_sent_at, :datetime
  add_column :users, :unconfirmed_email, :string
  add_index :users, :confirmation_token, unique: true
  # User.reset_column_information # Need for some types of updates, but not for update_all.
  execute("UPDATE users SET confirmed_at = NOW()")
end
def down
  remove_columns :users, :confirmation_token, :confirmed_at, :confirmation_sent_at
  remove_columns :users, :unconfirmed_email # Only if using reconfirmable
end
```
＊executeは、user情報を更新する際にconfirmed_atの値を設定するSQL文を直接発行
＊add_column,remove_columnなどのRailsがデフォルトで設定しているマイグレーションのメソッドだけではしたいことがしきれない場合、executeメソッドを使用して任意のSQLを実行できます。

3. マイグレーションを実行
4. letter_opener_webの設定
  - gem'letter_opener_web'をインストール
  - [config/environments/development.rb]に追記
```
config.action_mailer.default_url_options = { host: 'localhost', port: 3000 }
config.action_mailer.delivery_method = :letter_opener_web
```
  - [config/routes.rb]に追記
```
if Rails.env.development?
  mount LetterOpenerWeb::Engine, at: "/letter_opener"
end
```
5. すでにUserテーブルのレコードがある場合、エラー回避のため全て消去
```
$ rails c
$ User.delete_all
```
6. アプリ上でアカウントを作成すると、[http://localhost:3000/letter_opener]にメールが届くので、そのメールのリンクをクリックするとアカウントが作成される
7. 
# 疑問点
コントローラーのカスタマイズについて、どのようにカスタマイズができるのか？
