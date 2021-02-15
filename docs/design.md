# 設計

## `.roshi` ディレクトリ

- このディレクトリが存在する場所を `roshi` の管理下とする
- 中身
  - `.roshi/origin`: 元ディレクトリへのパス
  - `.roshi/origin-modtime.json`: 前回の `pull`, `push` 時の元ディレクトリのファイルの最終編集時刻
    - `pull` 時に更新
  - `.roshi/derive-modtime.json`: 前回の `pull`, `push` 時の管理下にあるファイルの最終編集時刻
    - `pull` 時に更新

## `.roshi.json` ファイル

- 用途的には `pull` と `push` のときに利用する
- `.roshi` ディレクトリと同じ階層にあるべき
  - origin, derive ともに全て列挙したとき, それぞれに重複すると見られるようなパターンは認められない (doubling)
    - 元ディレクトリのファイルと管理下のファイルを相互に同定しないといけないため
  - 各パターンについて正規表現なども認めない
  - 各パターンについてナンバリングがハイフンなしで連続してもマッチがやばいので認めない

## `.roshi.json` のバリデーションの流れ

- `.roshi.json` を読み込んで `map[string]string` を作る
- ループで key, value それぞれの組に対してバリデーションし, 問題のものがあれば終了
  - 連続するナンバリングや正規表現などの危ないパターンを弾く
  - 同じナンバリングの番号が複数含まれてしまっているパターンを弾く
  - key, value ともに順序を無視してまったく同じナンバリングのみを持たないものを弾く
  - この途中で key, value それぞれのスライスを用意する
- 各組のバリデーションが全て通ったあと, key, value のスライスそれぞれにダブる要素があるか確認し, もしあれば終了する
- ここまで来たら OK

## サブコマンド

便宜的に元ディレクトリのファイルを ofile, 管理下にある対応するファイルを dfile とする.

「changed」とは, `.roshi/*-modtime.json` に記録されている最終更新時刻から変化があったかどうかを指している. なお, 新規に作成された場合 (最終更新時刻が記録されていなかった場合) もこの「changed」にあてはまるものとする.

- `init path/to/source/directory`: 初期化
  - `.roshi` ディレクトリを作成
  - `.roshi/origin` が親にあれば失敗する
- `pull`: 元ディレクトリからパターンに合うファイルを取ってきて管理下の対応するパスに置く
  - `.roshi.json` を参照する (`map[*regexp.Regexp]string` を得る)
  - `.roshi/origin-modtime.json` を参照
  - `.roshi/derive-modtime.json` を参照
  - ofile changed
    - dfile exists
      - dfile changed => 上書き or create timestamped file (confirm)
      - dfile NOT changed => 上書き
    - dfile NOT exists => 上書き
  - ofile NOT changed => 放置
  - `.roshi/origin-modtime.json` を更新
  - `.roshi/derive-modtime.json` を更新
- `push`: 管理下にあるファイルを対応する元ディレクトリのパスに書き込む
  - `.roshi.json` を参照する (`map[*regexp.Regexp]string` を得る)
  - `.roshi/origin-modtime.json` を参照
  - `.roshi/derive-modtime.json` を参照
  - dfile changed
    - ofile exists
      - ofile changed => 上書き or 放置 (confirm)
      - ofile NOT changed => 上書き
    - ofile NOT exists => 上書き or 放置 (confirm)
  - dfile NOT changed => 放置
  - `.roshi/origin-modtime.json` を更新
  - `.roshi/derive-modtime.json` を更新
