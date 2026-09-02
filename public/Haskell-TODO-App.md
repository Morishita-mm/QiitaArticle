---
title: Haskell徹底解説：一歩ずつ作るTODOアプリ
tags:
  - Haskell
  - 関数型プログラミング
private: false
updated_at: '2026-02-10T21:41:50+09:00'
id: e980295a8f6ae58b15c7
organization_url_name: null
slide: false
ignorePublish: false
---
本記事は、[soupi.github.io/rfc/writing_simple_haskell/](https://soupi.github.io/rfc/writing_simple_haskell/) のHaskell TODOアプリに関するチュートリアルを参考に、日本語に翻訳し、より詳細な解説を加えたものです。

Haskellは「手順」を命じるのではなく、「データが何であるか」を定義する言語です。TODOアプリの機能をひとつずつ実装しながら、その作法を学んでいきましょう。

---

## Step 1: プログラムの骨格とIO

まずは、ユーザーに挨拶をして終わるだけの最小構成から始めます。

```haskell
module Todo where

main :: IO ()
main = do
  putStrLn "TODO app"
  putStrLn "Thanks for using this app."

```

### IOアクションと `do` 構文

* **`main :: IO ()`**: プログラムの入り口です。`IO` という型は、これが「現実世界を操作する処理（副作用）」であることを示します。
* **`do`**: 本来Haskellに順序はありませんが、`do` の中だけは例外的に上から順に実行されます。
* **インデント**: `do` の後の処理は、開始位置を垂直に揃える必要があります。

---

## Step 2: データの定義と「ループ」の代替（再帰による状態管理）

TODOリストの項目を管理するために、Haskellで独自の型を定義し、状態をどのように扱うかを見ていきます。Haskellには、伝統的な命令型言語のような `while` ループや変数の直接的な「更新」は存在しません。代わりに**再帰**を用いて状態を管理します。

```haskell
module Todo where -- モジュール宣言（省略）

-- (略) Step 1のコード

-- TODOアイテムの型を定義
type Item = String -- Item型はString型の別名
type Items = [Item] -- Items型はItem型のリストの別名

-- ユーザーとの対話を行うIOアクション
-- 現在のTODOアイテムのリストを受け取り、IOアクションを返す
interactWithUser :: Items -> IO ()
interactWithUser items = do
  putStrLn "コマンドを入力してください（例: add - タスク名, items, quit）:" -- プロンプト表示
  line <- getLine -- 標準入力から1行読み込み、その結果をlineに束縛するIOアクション
  if line == "quit" -- 入力が"quit"かどうかを判定
    then putStrLn "アプリケーションを終了します。またのご利用をお待ちしております！" -- 終了メッセージ表示
    else interactWithUser items -- "quit"でなければ、現在のitemsを引数に自分自身を再度呼び出し
```

### `type` と `data`：Haskellにおける型定義

*   **`type Item = String`**: `type` キーワードは、既存の型の**別名**を定義するために使われます。`Item` は単に `String` と同義であり、コンパイル時に `String` として扱われます。これにより、コードの意図が明確になり、可読性が向上します。
*   **`data`**: 後続のStepで登場する `data Command` のように、`data` キーワードは**新しいデータ型**を定義するために使われます。`data` 型は、複数のコンストラクタを持つことができ、異なる種類のデータを表現するのに利用されます。

### 変数の「書き換え」ではなく「再帰」で状態を回す

Haskellを含む純粋関数型言語では、**参照透過性（Referential Transparency）**が重要な特性です。これは、「同じ引数を渡せば常に同じ結果を返す」という関数の性質を指し、プログラムの推論やテストを容易にします。この性質を保つため、Haskellでは一度定義された変数の値を後から変更する（ミュータブルな変数）ことはできません。

では、TODOリストのような「状態」を持つアプリケーションはどのように実現するのでしょうか？その答えが**再帰**です。

*   **`interactWithUser items`**: `interactWithUser` 関数は、現在のTODOリスト (`items`) を引数として受け取ります。ユーザーがコマンドを入力し、もしそのコマンドが終了（`quit`）でなければ、`interactWithUser` 関数は**新しい `items` の状態（追加や削除が行われた後）を引数として、自分自身を再度呼び出します。**これにより、あたかもループしているかのように連続した処理を実現しながら、各呼び出しでは純粋な関数として振る舞い、変数の直接的な書き換えを避けています。

*   **`line <- getLine`**: ここで登場する `<-` 演算子は、`IO` アクション（`getLine`）を実行し、その**結果の値（この場合はユーザーが入力した `String`）を純粋な変数 `line` に「束縛」**します。`getLine` 自体は `IO String` 型、つまり副作用を伴うアクションですが、`<-` を使うことでその結果の `String` だけを取り出し、`do` ブロック内で純粋な値として扱えるようになります。これは、Haskellが副作用と純粋な計算を分離するメカニズムの核心部分です。

このステップでは、Haskellが変数の不変性をどのように維持し、再帰と`<-`演算子を組み合わせて状態管理や副作用の結果を扱っているかを学びました。

---

## Step 3: コマンドの「パース」とパターンの分離（強力なパターンマッチ）

ユーザーからの入力文字列をアプリケーションが理解できる「コマンド」に変換するプロセスを**パース（解析）**と呼びます。Haskellでは、このパース処理に**代数的データ型（Algebraic Data Type: ADT）**と**パターンマッチ**という強力な機能を活用します。

```haskell
module Todo where -- モジュール宣言（省略）

-- (略) Step 1, 2のコード

-- コマンドを表現する代数的データ型
-- ユーザーが入力する可能性のあるコマンドとその引数を型として定義
data Command
  = Quit           -- アプリケーション終了コマンド（引数なし）
  | DisplayItems   -- TODOリスト表示コマンド（引数なし）
  | AddItem String -- TODOアイテム追加コマンド（String型の引数を持つ）
  | Done Int       -- TODOアイテム完了コマンド（Int型のインデックス引数を持つ）
  | Help           -- ヘルプ表示コマンド（引数なし）

-- 文字列をCommand型にパースする純粋関数
-- 失敗の可能性があるため、Either型で結果を表現
parseCommand :: String -> Either String Command
parseCommand line = case words line of -- 入力文字列を単語のリストに分割し、パターンマッチ
  ["quit"]                      -> Right Quit         -- "quit"のみのリストにマッチ
  ["items"]                     -> Right DisplayItems -- "items"のみのリストにマッチ
  ["help"]                      -> Right Help         -- "help"のみのリストにマッチ
  "add" : "-" : itemWords       -> Right (AddItem (unwords itemWords)) -- "add", "-", 残りの単語にマッチ
  ["done", idxStr]              -> -- "done", インデックス文字列にマッチ
    if all (`elem` "0123456789") idxStr -- インデックス文字列が全て数字で構成されているかチェック
      then Right (Done (read idxStr)) -- 数字ならIntに変換してDoneコンストラクタでラップ
      else Left "Error: Invalid index format for 'done' command. Please provide a number." -- 数字でなければエラー
  _                             -> Left "Error: Unknown command. Type 'help' for available commands." -- 上記のどのパターンにもマッチしない場合
```

### 代数的データ型 `data Command` とパターンマッチ

*   **`data Command = Quit | DisplayItems | AddItem String | Done Int | Help`**:
    `data` キーワードで定義された `Command` は**代数的データ型**の一例です。これは、複数の異なる可能性を持つ型を表現するのに非常に便利です。
    *   `Quit`, `DisplayItems`, `Help` は引数を持たない**データコンストラクタ**で、特定のコマンド自体を表します（列挙型のようなもの）。
    *   `AddItem String` は `String` 型の引数を受け取るデータコンストラクタです。これにより、「追加するアイテム」という情報をコマンドに含めることができます。
    *   `Done Int` も同様に `Int` 型の引数を受け取ります。
    このように、`Command` 型の値を検査することで、それがどの種類のコマンドであるか、そしてどのような情報を持っているかを、型安全な方法で知ることができます。

*   **`case ... of` とパターンマッチ**:
    Haskellの**パターンマッチ**は、データ構造の形に基づいて処理を分岐させる強力なメカニズムです。`parseCommand` 関数では、`case words line of ...` を用いて、入力文字列を `words` 関数で分割したリストに対してパターンマッチを行っています。
    *   `["quit"]`: `["quit"]` という正確なリストにマッチします。
    *   `"add" : "-" : itemWords`: これはリストのコンス演算子 (`:`) を用いたパターンです。
        *   リストの最初の要素が `"add"` であること。
        *   2番目の要素が `"-"` であること。
        *   残りの要素（リスト）が `itemWords` という変数に束縛されること。
        という条件にマッチします。`itemWords` には、例えば `"add - My new task"` という入力であれば `["My", "new", "task"]` が入ります。
    *   `_`: これは「ワイルドカード」パターンと呼ばれ、上記のどのパターンにもマッチしなかった場合に捕捉する役割を果たします。これにより、未定義のコマンドに対して適切なエラーメッセージを返すことができます。

### `Either String Command`：失敗する可能性のある計算の表現

*   **`Either a b`**: `Either` 型は、Haskellにおける非常に一般的なエラーハンドリングのパターンです。これは、「成功した値」と「失敗した値」のいずれかを保持できる型です。
    *   `Left a`: 計算が失敗した場合に、エラーメッセージなどの `a` 型の値を保持します。
    *   `Right b`: 計算が成功した場合に、結果の `b` 型の値を保持します。
    `parseCommand` 関数では、`Left String` でエラーメッセージ（`String`）を返し、`Right Command` でパースされた `Command` 型の値を返しています。これにより、関数が返す型を見ただけで「この関数は失敗する可能性がある」と明確に伝えられ、呼び出し側はエラーケースを適切に処理するよう強制されます。

*   **`words` と `unwords`**:
    *   `words :: String -> [String]`: スペースで区切られた文字列を `String` のリストに変換します。
    *   `unwords :: [String] -> String`: `String` のリストをスペースで区切られた単一の文字列に結合します。
    これらの関数は、コマンドのパースにおいて非常に便利です。

このステップでは、Haskellが代数的データ型とパターンマッチをどのように組み合わせて構造化されたデータを扱い、`Either` 型によって失敗の可能性を型安全に表現しているかを学びました。

---

## Step 4: TODOリストの操作（純粋なロジックとHaskellの強力な機能）

Haskellでは、TODOリストの追加や表示といったロジックを、副作用のない**純粋な関数**として記述できます。これにより、各関数は独立してテストしやすく、プログラム全体の信頼性が向上します。ここではHaskellの強力な機能である**遅延評価**と**高階関数**がどのように活用されるかを見ていきましょう。

```haskell
module Todo where -- モジュール宣言（省略）

-- (略) Step 1, 2, 3のコード

-- TODOアイテムをリストに追加する純粋関数
-- 新しいアイテムを既存のアイテムリストの先頭に追加し、新しいリストを返す
addItem :: Item -> Items -> Items
addItem item items = item : items -- `(:)`はリストの先頭に要素を追加する演算子

-- TODOリストの内容を整形された文字列として返す純粋関数
-- 各アイテムに番号を付けて表示
displayItems :: Items -> String
displayItems items =
  let
    -- displayItemはインデックスとアイテムを受け取り、「インデックス - アイテム」形式の文字列を返すローカル関数
    displayItem :: Int -> Item -> String
    displayItem index item = show index ++ " - " ++ item

    reversedList = reverse items -- 最新のアイテムをリストの先頭に表示するため、リストを逆順にする
    displayedItemsList = zipWith displayItem [1..] reversedList -- displayItem関数を使い、[1..]（無限リスト）と逆順リストを結合
  in
    unlines displayedItemsList -- 各行を改行で結合して単一の文字列を返す

-- 指定されたインデックスのアイテムを削除する純粋関数
-- 削除に失敗する可能性があるため、Either型で結果を表現
removeItem :: Int -> Items -> Either String Items
removeItem targetIndex allItems =
    -- ユーザーに表示されるインデックスは1から始まる逆順なので、実際のリストインデックスに変換
    let actualIndex = length allItems - targetIndex
    in
    impl actualIndex allItems -- 実際の削除処理はヘルパー関数`impl`に委譲
  where
    -- implは実際の0-basedインデックスとリストを受け取り、アイテムを削除するヘルパー関数
    impl :: Int -> Items -> Either String Items
    impl index items =
      case (index, items) of
        (0, _ : rest) -> Right rest -- インデックス0にマッチしたら、その要素をスキップし残りのリストを返す
        (_, [])       -> Left "Error: Index out of bounds. The item does not exist." -- 空のリストでインデックスが0でなければエラー
        (n, x : rest) -> -- n > 0 かつリストが空でない場合
          case impl (n - 1) rest of -- 残りのリストに対して再帰的に削除を試みる
            Right newRest -> Right (x : newRest) -- 成功したら現在の要素xを先頭に追加し新しいリストを構築
            Left err      -> Left err            -- 失敗したらエラーをそのまま伝播
```

### 純粋な関数によるデータ操作

*   **`addItem :: Item -> Items -> Items`**:
    `addItem` 関数は、`Item` (String) と `Items` ([String]) を受け取り、新しい `Items` を返します。重要なのは、元の `items` リストを**変更する**のではなく、新しい `items` リストを**生成して返す**という点です。Haskellでは、リストの先頭に要素を追加する `(:)` 演算子が非常に効率的です。

*   **`displayItems :: Items -> String`**:
    この関数は、`Items` リストを受け取り、整形された `String` を返します。注目すべきは、Haskellのいくつかの強力な機能が組み合わされている点です。

    *   **`let in` 構文**: `let displayItem index item = ... in ...` は、関数内でローカルな関数や変数を定義するために使われます。これにより、関数の可読性とモジュール性が向上します。
    *   **`[1..]` (無限リスト) と遅延評価**: `[1..]` は `1, 2, 3, ...` と続く無限の整数のリストを表します。Haskellの**遅延評価**（必要になるまで計算を実行しない戦略）のおかげで、このような無限リストを安全に扱うことができます。`zipWith` が必要とする分だけ要素が生成されるため、メモリが無限に消費されることはありません。
    *   **`zipWith :: (a -> b -> c) -> [a] -> [b] -> [c]` (高階関数)**:
        `zipWith` は、2つのリストの対応する要素に関数を適用し、その結果を新しいリストとして返す**高階関数**です。ここでは、`displayItem` 関数を `[1..]` と `reversedList` の各要素のペアに適用し、各TODOアイテムの表示文字列を生成しています。
    *   **`reverse items`**: `displayItems` で `reverse items` を行っているのは、`addItem` で常にリストの先頭に新しいアイテムを追加しているためです。これにより、最新のアイテムがリストの末尾（=表示時に一番下の行）に来るように調整しています。

*   **`removeItem :: Int -> Items -> Either String Items`**:
    `removeItem` 関数は、指定されたインデックスのアイテムを削除し、新しいリストを返します。この関数も元のリストを変更せず、新しいリストを生成します。

    *   **再帰によるリスト操作**: Haskellでリストから特定の要素を削除する典型的な方法は、再帰を用いることです。ヘルパー関数 `impl` が定義されており、インデックスが `0` になったときにその要素をスキップし、それ以外の場合は現在の要素 (`x`) を保持しつつ残りのリスト (`rest`) に対して再帰的に処理を続けます。
    *   **インデックスの変換**: ユーザーが入力する `targetIndex` は「1から始まる、表示上の逆順インデックス」なので、これをHaskellの内部的な「0から始まる、正順インデックス」に変換する必要があります (`actualIndex = length allItems - targetIndex`)。
    *   **`Either` 型によるエラーハンドリング**: `impl` 関数内でインデックスが範囲外である場合（空のリストに対して削除を試みるなど）には `Left` でエラーメッセージを返し、呼び出し元にエラーを伝播させています。

このステップでは、Haskellが純粋な関数、遅延評価、高階関数、そして再帰をどのように組み合わせて、効率的かつ堅牢なデータ操作ロジックを構築しているかを学びました。

---

## Step 5: 全てを繋ぎ合わせる（完成）とHaskellプログラミングのまとめ



これまでのステップで、Haskellの基本的な要素、つまり `IO` モナド、`do` 構文、型定義 (`type`, `data`)、パターンマッチ、`Either` 型によるエラーハンドリング、純粋な関数によるデータ操作、遅延評価、高階関数、そして再帰を学びました。Haskellの設計哲学である**純粋なロジックと副作用の分離**が、このTODOアプリのコードにいかに反映されているかを見ていきましょう。



`interactWithUser` 関数が、`IO` を通じてユーザーからの入力を受け取り、純粋な関数である `parseCommand`、`addItem`、`displayItems`、`removeItem` を呼び出すことで、アプリケーションのロジックが安全かつ効率的に実行されています。



### TODOアプリ 全文コード



```haskell

-- Todo.hs
module Todo where

-- 1. 型の定義
-- TODOアイテムは文字列として扱う
type Item = String
-- TODOアイテムのリスト
type Items = [Item]

-- ユーザーからのコマンドを表現する代数的データ型
data Command
  = Quit           -- アプリケーション終了
  | DisplayItems   -- アイテム表示
  | AddItem String -- アイテム追加（引数として追加するアイテム文字列を持つ）
  | Done Int       -- アイテム完了（引数として完了するアイテムのインデックスを持つ）
  | Help           -- ヘルプ表示

-- 2. メインエントリポイント
-- プログラムの開始点。IO () 型であることから、副作用を伴う処理であることを示す。
main :: IO ()
main = do
  putStrLn "TODO appへようこそ！"
  putStrLn "このHaskell製TODOアプリでタスクを管理しましょう。"
  interactWithUser [] -- 初期状態は空のTODOリストでユーザー対話を開始
  putStrLn "Thanks for using this app."

-- 3. ユーザー対話 (IO層)
-- 現在のTODOアイテムのリスト (Items) を受け取り、ユーザーからのコマンドに応じて処理を進める
interactWithUser :: Items -> IO ()
interactWithUser items = do
  putStrLn "\nコマンドを入力してください（help で一覧表示）:"
  line <- getLine -- ユーザーからの入力を読み込むIOアクション。結果を 'line' に束縛。
  case parseCommand line of -- 入力された文字列をCommand型にパース
    Right Help -> do
      putStrLn "利用可能なコマンド:"
      putStrLn "  help               - このヘルプを表示"
      putStrLn "  quit               - アプリケーションを終了"
      putStrLn "  items              - 現在のTODOリストを表示"
      putStrLn "  add - <アイテム名> - 新しいTODOアイテムを追加"
      putStrLn "  done <インデックス> - 指定されたインデックスのTODOアイテムを完了（削除）"
      interactWithUser items -- ヘルプ表示後、同じアイテムリストで対話を継続

    Right DisplayItems -> do
      putStrLn "現在のTODOリスト:"
      putStrLn (displayItems items) -- 純粋関数で整形されたリストを表示
      interactWithUser items -- リスト表示後、同じアイテムリストで対話を継続

    Right (AddItem item) -> do
      let newItems = addItem item items -- 純粋関数で新しいアイテムリストを生成
      putStrLn $ "アイテム '" ++ item ++ "' を追加しました。"
      interactWithUser newItems -- 新しいアイテムリストで対話を継続

    Right (Done index) -> do
      case removeItem index items of -- 純粋関数でアイテム削除を試みる
        Left errMsg -> do
          putStrLn $ "エラー: " ++ errMsg
          interactWithUser items -- エラーメッセージ表示後、同じアイテムリストで対話を継続

        Right newItems -> do
          putStrLn $ show index ++ "番のアイテムを完了しました。"
          interactWithUser newItems -- 削除成功後、新しいアイテムリストで対話を継続

    Right Quit -> putStrLn "アプリケーションを終了します。" -- 終了メッセージを表示し、対話ループを終了

    Left errMsg -> do
      putStrLn $ "エラー: " ++ errMsg
      interactWithUser items -- エラーメッセージ表示後、同じアイテムリストで対話を継続

-- 4. コマンド解析 (純粋関数)
-- 文字列を受け取り、Command型またはエラーメッセージを返す。
parseCommand :: String -> Either String Command
parseCommand line = case words line of -- 入力文字列を単語リストに分割し、パターンマッチ
  ["quit"]                      -> Right Quit
  ["items"]                     -> Right DisplayItems
  ["help"]                      -> Right Help
  "add" : "-" : itemWords       -> Right (AddItem (unwords itemWords)) -- "add - <アイテム名>" のパターン
  ["done", idxStr]              -> -- "done <インデックス>" のパターン
    if all (`elem` "0123456789") idxStr -- インデックスが数字のみか確認
      then Right (Done (read idxStr)) -- 数字であればIntに変換
      else Left "インデックスの形式が不正です。数値を入力してください。"
  _                             -> Left "不明なコマンドです。'help'で利用可能なコマンドを確認してください。"

-- 5. データ操作ロジック (純粋関数)
-- 新しいアイテムをリストの先頭に追加する
addItem :: Item -> Items -> Items
addItem item items = item : items

-- アイテムリストを整形して文字列として返す
displayItems :: Items -> String
displayItems items =
  let
    displayItem :: Int -> Item -> String
    displayItem index item = show index ++ " - " ++ item

    reversedList = reverse items -- 最新のアイテムが下に来るように逆順に
    displayedItemsList = zipWith displayItem [1..] reversedList
  in
    if null items
      then "  (TODOアイテムはまだありません。)"
      else unlines displayedItemsList

-- 指定されたインデックスのアイテムをリストから削除する
removeItem :: Int -> Items -> Either String Items
removeItem targetIndex allItems =
    let actualIndex = length allItems - targetIndex -- 表示インデックスから実際のリストインデックスに変換
    in
    if targetIndex <= 0 || targetIndex > length allItems
      then Left "指定されたインデックスのアイテムは存在しません。"
      else impl actualIndex allItems
  where
    impl :: Int -> Items -> Either String Items
    impl index items =
      case (index, items) of
        (0, _ : rest) -> Right rest -- 0番目の要素をスキップ
        (_, [])       -> Left "インデックスが範囲外です。" -- このパスには到達しないはずだが念のため
        (n, x : rest) -> case impl (n - 1) rest of
                           Right newRest -> Right (x : newRest)
                           Left err      -> Left err
```



### アプリケーションの実行方法

このTODOアプリケーションのコードを `Todo.hs` というファイル名で保存してください。

1.  **コンパイル**: HaskellのコンパイラであるGHC (Glasgow Haskell Compiler) を使用してコンパイルします。

    ```bash
    ghc Todo.hs
    ```

    これにより、`Todo` という実行可能ファイルが生成されます。



2.  **実行**: 生成された実行可能ファイルを実行します。

    ```bash
    ./Todo
    ```



### アプリケーションの使用例



```
❯ runghc Main.hs
TODO appへようこそ！
このHaskell製TODOアプリでタスクを管理しましょう。

コマンドを入力してください（help で一覧表示）:
add - Haskellの勉強
アイテム 'Haskellの勉強' を追加しました。

コマンドを入力してください（help で一覧表示）:
add - 記事を書く
アイテム '記事を書く' を追加しました。

コマンドを入力してください（help で一覧表示）:
items
現在のTODOリスト:
1 - Haskellの勉強
2 - 記事を書く


コマンドを入力してください（help で一覧表示）:
done 1
1番のアイテムを完了しました。

コマンドを入力してください（help で一覧表示）:
items
現在のTODOリスト:
1 - 記事を書く


コマンドを入力してください（help で一覧表示）:
help
利用可能なコマンド:
  help               - このヘルプを表示
  quit               - アプリケーションを終了
  items              - 現在のTODOリストを表示
  add - <アイテム名> - 新しいTODOアイテムを追加
  done <インデックス> - 指定されたインデックスのTODOアイテムを完了（削除）

コマンドを入力してください（help で一覧表示）:
quit
アプリケーションを終了します。
Thanks for using this app.
```



### まとめ：Haskellプログラミングの核心



このTODOアプリの構築を通じて、Haskellの以下の重要な概念と利点を体験できたことでしょう。



*   **型システム**: `IO` モナド、`Either` 型、代数的データ型など、Haskellの強力な型システムがプログラムの安全性と堅牢性をどのように高めるか。

*   **純粋性と副作用の分離**: 関数が入力に対して常に同じ出力を返す「純粋性」を保ちつつ、IOモナドを通じて副作用を安全に管理する方法。

*   **宣言的プログラミング**: 「どう計算するか」ではなく「何を計算するか」を記述することで、コードがより簡潔で分かりやすくなること。

*   **パターンマッチ**: 複雑なデータ構造を直感的に分解し、処理を分岐させる強力なメカニズム。

*   **遅延評価と高階関数**: 無限リストや関数を引数として扱う高階関数が、柔軟で表現豊かなコードを可能にすること。

*   **再帰**: ループの代わりに再帰を用いることで、状態の変更を伴わない（イミュータブルな）プログラミングスタイルを確立する方法。



Haskellは、これらの概念を通じて、バグの少ない、保守しやすい、そして並列処理にも適したコードを書くことを可能にします。このチュートリアルが、Haskellの奥深い世界への第一歩となることを願っています。ぜひ、このTODOアプリを拡張したり、他のアプリケーションをHaskellで実装したりしてみてください。



---

