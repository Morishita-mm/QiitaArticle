---
title: Rustの並行処理
tags:
  - Rust
  - メモ
  - 初心者
  - 並行処理
private: false
updated_at: '2025-11-02T17:55:48+09:00'
id: 47217692a007f6a1299f
organization_url_name: null
slide: false
ignorePublish: false
---
> 自分用のメモの側面を含みますので、読みづらい可能性があります。ご留意ください。

### はじめに
並行処理は「速くする」ための道具ですが、常に早くなるわけではありません。ここではRustを使い、単純な処理と待機時間の長い処理（擬似的にsleep）を並行化した時に実行時間を比較し、「なぜ遅くなる場合があるのか」を考えます。

Rustの導入や詳しいクレーと解説は省略します。Rustの基礎は[公式チュートリアル](https://doc.rust-jp.rs/book-ja/title-page.html)が非常にまとまっています。
### 目標
以下の様なタスクを想定します：
- `1..=N`の配列から要素を取り出す
- 処理を施す（単純／複雑の2パターン）
- 条件を満たした値を集める
単純な処理は「偶奇判定」、複雑な処理は「一定時間待機してから偶奇判定」とします。並行処理（マルチスレッド）と逐次処理（シングルスレッド）で時間を比較します。

### 準備
測定や処理の骨格は以下を用います。
```rust
// 使用するクレート
use std::{
  sync::Mutex,
  thread,
  time::{Duration, Instant},  // 実行時間計測用
};

const WAIT_MILLIS: u64 = 1000;   // 待機時間（複雑な処理の擬似負荷）

// 複雑な処理（待機→偶奇判定）
fn complex_process(n: u64) -> bool {
  thread::sleep(Duration::from_millis(WAIT_MILLIS));
  n % 2 == 0
}

// 単純な処理（偶奇判定のみ）
fn simple_process(n: u64) -> bool {
    n % 2 == 0
}
```

**マルチスレッド版**
```rust
const IS_COMP: bool = false;        // 複雑な処理をならtrue、単純な処理ならfalse
const N_THREADS: usize = 10;        // スレッド数
const NUMS: u64 = if IS_COMP { 10 } else { 1_000_000 };       // 処理回数

fn main() {
    let start = Instant::now(); // 計測開始

    let ans = Mutex::new(Vec::new());
    let nums: Mutex<Vec<u64>> = Mutex::new((1..=NUMS).collect());

    thread::scope(|s| {
        for _ in 0..N_THREADS {
            s.spawn(|| {
                loop {
                    let num_opt = nums.lock().unwrap().pop();
                    let num = match num_opt {
                        Some(n) => n,
                        None => break,
                    };

                    let keep = if IS_COMP {
                        complex_process(num)
                    } else {
                        simple_process(num)
                    };

                    if keep {
                        ans.lock().unwrap().push(num);
                    }
                }
            });
        }
    });

    if let Ok(final_ans) = ans.into_inner() {
        // 計測終了
        let end = start.elapsed();
        // 計測時間の出力
        let pre = if IS_COMP { "complex" } else { "simple" };
        println!("{}: multi: {}ms", pre, end.as_millis());
    }
}
```

**シングルスレッド版**
```rust
const IS_COMP: bool = false;
const NUMS: u64 = if IS_COMP { 10 } else { 1000000 };

fn main() {
    let start = Instant::now(); // 計測開始

    let mut nums: Vec<u64> = (1..=NUMS).collect();
    let mut ans = Vec::new();

    loop {
        let num = match nums.pop() {
            Some(n) => n,
            None => break,
        };

        let keep = if IS_COMP {
            complex_process(num)
        } else {
            simple_process(num)
        };
        
        if keep {
            ans.push(num);
        }
    }

    // 計測終了
    let end = start.elapsed();
    // 計測時間の出力
    println!("singlethread: {}ms", end.as_millis());
}
```

### 実行結果
環境差はあるので、パラメータを変えつつ試してみてください。筆者の環境では概ね以下の傾向が見られました。
- 単純な処理（偶奇判定のみ）
    - スレッド数を増やすほど遅くなる（1→2→3→5→10→20と悪化）
- 複雑な処理（`sleep`で待機後に判定）
    - スレッド数を増やすほど速くなる（1→2→3→5→10と改善）
    - 10と20では差がほぼ出ない（仕事量が10に対してスレッド数が過剰）

| 処理内容 | スレッド数 | 平均時間 (ms) |
|----------|------------|---------------|
| 単純な処理 | 01 | 16.700 |
| 複雑な処理 | 01 | 10017.200 |
| 単純な処理 | 02 | 79.500 |
| 複雑な処理 | 02 | 5018.900 |
| 単純な処理 | 03 | 84.400 |
| 複雑な処理 | 03 | 4007.000 |
| 単純な処理 | 05 | 94.100 |
| 複雑な処理 | 05 | 2006.300 |
| 単純な処理 | 10 | 140.900 |
| 複雑な処理 | 10 | 1002.100 |
| 単純な処理 | 20 | 210.800 |
| 複雑な処理 | 20 | 1001.700 |

![単純な処理 と 複雑な処理.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371529/1a510be8-0fb5-4a18-945c-9005e23d03f7.png)

今回のポイントは「**単純処理は並行で遅く、待機のある複雑処理は並行で速い**」という対照的な結果にあります。

### なぜこうなるのか❓
結論から言うと、**並行処理によるオーバーヘッドが原因**です。
並行処理には「オーバーヘッド」があり、仕事が小さいほどそのコストが支配的になります。一方で「待機が大部分」を占める処理は、待機の同時進行によって全体の実行時間を短縮できます。

- **単純処理が遅くなる理由**
    - スレッド生成・スケジューリングのコスト
    - 共有`Vec`からの取り出しや結果格納のための`Mutex`競合
    - 仕事（偶奇判定）が極小で、並行化のメリットよりオーバーヘッドが勝つ
- **複雑処理（sleep）で速くなる理由**
    - 各スレッドはCPUをほぼ使わず「待っている」時間が長い
    - 待機を同時に進められるため、全体の経過時間が短くなる
    - ただし仕事量（NUMS）が少ないと、スレッド数を増やしても頭打ちになる（過剰並列）

これを裏付ける法則として、並列化による理論的な加速限界を求める法則である**アムダールの法則**が知られています
基本形は以下のとおりです。
$$ 速度向上（Speedup） ≈ \frac{1}{(1 - P) + \frac{P}{N}} $$
- **P**：並列化可能な割合
- **N**：並列度（スレッド数など）

並列化に伴うオーバーヘッドを加えると、以下の様に拡張できます。
$$ 速度向上（Speedup） ≈  \frac{1}{(1 - P) + \frac{P}{N}+H} $$
- **H**：並列化によって増える余分な処理時間の割合（同期や管理のコスト）

この法則から以下のことが見えてきます：
- 単純処理はPが高く見えても、H（オーバーヘッド）が相対的に大きく、総時間が増える
- 待機のある複雑処理はPが高く、かつHが相対的に小さくなりやすいため、並行性の恩恵が出る

### まとめ
並行処理は「常に速い」わけではなく、オーバーヘドが支配的にある場面ではむしろ遅くなります。
どの様な時に並行処理が有効なのか考えながら実装してみましょう。
また、今回は非常に単純な並行処理を行いましたが、他にも様々な並行処理のためのクレートがあります。
興味を持った人はぜひそちらについても調べてみてください。

### おまけ
筆者が実際に計測を行なったプログラムです。
実行してみたい方はこちらをコピペしてもらえれば実行できます。

<details><summary>サンプルプログラム</summary>

```rust
use std::{
    collections::HashMap,
    sync::Mutex,
    thread,
    time::{Duration, Instant}, // 実行時間計測用
};

const WAIT_MILLIS: u64 = 10; // 待機時間

// 複雑な処理をする関数（擬似的にスリープで再現）
fn complex_process(n: u64) -> bool {
    thread::sleep(Duration::from_millis(WAIT_MILLIS));
    n % 2 == 0
}

// 単純な処理をする関数（奇数ならfalse、偶数ならtrueを返す）
fn simple_process(n: u64) -> bool {
    n % 2 == 0
}

const N_ITEROS: u16 = 10;

// 並行処理
fn main() {
    let mut result: HashMap<(bool, i32), Vec<u128>> = HashMap::new();
    let thread_list = vec![1, 2, 3, 5, 10, 20];
    let comp_list = vec![true, false];

    for is_comp in comp_list {
        for n_threads in &thread_list {
            for _ in 0..N_ITEROS {
                let res = if *n_threads == 1 {
                    single(is_comp)
                } else {
                    multi(is_comp, *n_threads)
                };
                let key = (is_comp, *n_threads);
                result
                    .entry(key)
                    .or_insert_with(Vec::new)
                    .push(res.as_millis());
            }
        }
    }

    show_result(&result);
}

fn multi(is_comp: bool, n_threads: i32) -> Duration {
    let nums: u64 = if is_comp { 10 } else { 1000000 }; // 何回処理を行うか
    let start = Instant::now(); // 計測開始

    let ans = Mutex::new(Vec::new());
    let nums: Mutex<Vec<u64>> = Mutex::new((1..=nums).collect());

    thread::scope(|s| {
        for _ in 0..n_threads {
            s.spawn(|| {
                loop {
                    let num_opt = nums.lock().unwrap().pop();
                    let num = match num_opt {
                        Some(n) => n,
                        None => break,
                    };

                    let keep = if is_comp {
                        complex_process(num)
                    } else {
                        simple_process(num)
                    };

                    if keep {
                        ans.lock().unwrap().push(num);
                    }
                }
            });
        }
    });

    if let Ok(final_ans) = ans.into_inner() {
        // 計測終了
        let end = start.elapsed();
        return end;
    } else {
        panic!("Something went wrong");
    }
}

fn single(is_comp: bool) -> Duration {
    let nums: u64 = if is_comp { 10 } else { 1000000 }; // 何回処理を行うか

    let start = Instant::now(); // 計測開始

    let mut nums: Vec<u64> = (1..=nums).collect();
    let mut ans = Vec::new();

    loop {
        let num = match nums.pop() {
            Some(n) => n,
            None => break,
        };

        let keep = if is_comp {
            complex_process(num)
        } else {
            simple_process(num)
        };

        if keep {
            ans.push(num);
        }
    }

    return start.elapsed();
}

fn show_result(result: &HashMap<(bool, i32), Vec<u128>>) {
    let mut keys: Vec<&(bool, i32)> = result.keys().collect();

    keys.sort_by_key(|k| (k.1, k.0));

    println!(
        "--- 実行時間計測結果 (平均 {} 回 / スレッド数昇順) ---",
        N_ITEROS
    );
    println!("| 処理内容 | スレッド数 | 平均時間 (ms) |");
    println!("|----------|------------|---------------|");

    for key in keys {
        let (is_comp, n_threads) = *key;
        let times = result.get(key).unwrap();

        // 合計と平均の計算
        let sum: u128 = times.iter().sum();
        let count = times.len() as u128;

        let average_ms = if count > 0 {
            (sum as f64) / (count as f64)
        } else {
            0.0
        };

        let comp_label = if is_comp {
            "複雑な処理"
        } else {
            "単純な処理"
        };

        println!("| {} | {:02} | {:5.3} |", comp_label, n_threads, average_ms);
    }
    println!("--------------------------------------");
}
```
</details>
