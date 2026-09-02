---
title: AIへの指示は「ハイブリッド」が最強：文脈はMarkdown、制御はYAMLで書く「関心事の分離」
tags:
  - 生成AI
  - LLM
  - プロンプトエンジニアリング
private: false
updated_at: '2025-12-06T01:50:42+09:00'
id: 11b7e408302d0fa43b70
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

「AIにJSONで出力させて、かつエラー処理も厳密にさせたい」
そう思ってMarkdownで長い指示書を書いたのに、AIが微妙に指示を無視したり、挨拶文を混ぜてきたりしてイライラした経験はありませんか？

実は、最新の研究では<strong>「AIはMarkdown形式のデータを読むのが得意」</strong>という結果が出ています。しかし一方で、システム開発の現場では<strong>「YAMLのような厳密なデータ形式の方が制御しやすい」</strong>という事実もあります。

この矛盾をどう解決すべきか？
本記事では、両者のいいとこ取りをする<strong>「ハイブリッド・プロンプト（Hybrid Prompting）」</strong>という手法を提案します。

それは、プロンプトを「文章」ではなく「アーキテクチャ」として捉え、**「文脈（Context）」と「制御（Control）」を分離する**アプローチです。

## 第1章：Markdownの功罪と「読み取り精度」の真実

### 誰もMarkdownを捨てられない理由

最近の論文（金融ドメインにおけるLLMのフォーマット比較など）において、数値理解や情報抽出タスクでは、YAMLよりもMarkdownやJSONの方が精度が高いという結果が報告されています[^1]。
LLMの学習データの大半はWeb上のテキスト（Markdown/HTML）であるため、AIにとって「文章を読んで文脈を理解する」にはMarkdownが最も適したインターフェースなのです。

### しかし、Markdownは「仕様書」には向かない

一方で、Markdownには致命的な弱点があります。それは<strong>「構造の強制力が弱い」</strong>ことです。
「出力は50文字以内」「厳格なJSON形式」といった<strong>制約（Constraints）</strong>を自然言語の箇条書きで伝えると、AIはそれを「推奨事項」程度に受け取ってしまうことがあります。

つまり、**「マニュアル（読みやすさ）」としてはMarkdownが優秀ですが、「設計図（厳密さ）」としては機能不足**なのです。

## 第2章：解決策としての「ハイブリッド・プロンプト」

そこで提案するのが、以下の構成です。

> **The Hybrid Principle:**
>
>   * **Context (背景・データ):** Markdownで記述する（AIの読解力を最大化）
>   * **Control (指示・制約):** YAMLで記述する（AIの実行精度を最大化）

これらを一つのプロンプト内で明確にセクション分けします。

### プロンプト構成例

````md
# 1. Context (Input Data)
## ユーザーの状況
現在、Python学習中の初心者が、以下のコードのバグに悩んでいます。
## 対象コード
```python
def example(): pass
```

-----

# 2. System Configuration (Instructions)

## ここはAIを「制御」したいのでYAMLで書く

```yaml
system_profile:
  role: &teacher
    name: Enthusiastic Python Tutor
    tone: Encouraging but precise

instruction_schema:
  objective: Debug and Refactor
  constraints:
    - strict_output: true  # 余計なお喋りを禁止
    - max_severity: High
  
output_interface:
  format: JSON
  schema:
    type: object
    properties:
      fixed_code: { type: string }
      explanation: { type: string, maxLength: 200 }

```
````

このように記述することで、AIは<strong>「Markdownパートで文脈を深く理解」</strong>しつつ、<strong>「YAMLパートで定義された厳格なルールに従って出力」</strong>します。

## 第3章：なぜ「制御（Control）」はYAMLなのか？

「制御」パートにJSONやXMLではなく、あえてYAMLを採用するのには、エンジニアリング観点での明確な理由があります。

### 1. Git Diffでの「変更管理」が劇的に楽になる
プロンプトをGitで管理する場合、Markdown（自然言語）の修正はDiffが汚くなりがちです。しかしYAMLであれば、変更は「値の更新」として表現されます。

**Markdownの場合（Diffが読みづらい）:**
```diff
- ユーザーに対して、優しく丁寧に、かつ簡潔に回答してください。
+ ユーザーに対して、セキュリティの観点から厳格に回答してください。
````

**YAMLの場合（意図が明確）:**

```diff
  role:
-   tone: Gentle and Concise
+   tone: Strict and Security-Focused
```

チーム開発において、「プロンプトのどのパラメータを調整したのか」が一目瞭然になります。

### 2\. コメントによる「意図」の注入

JSONは構造化されていますが、コメントが書けません。
プロンプトエンジニアリングでは、「なぜこの制約が必要なのか」という**Few-shot的なニュアンス**をコメントで補足することで、AIのハルシネーションを抑制できます。YAMLはこの要件を満たす唯一の軽量フォーマットです。

### 3\. AI to AI 通信のプロトコルとして

MetaGPT[^2]などのマルチエージェント研究において、エージェント間の連携（SOP: 標準作業手順）を構造化することの重要性が示唆されています。
「Architect AI」が仕様をYAMLで定義し、「Coder AI」がそのYAMLを読み込んで実装する。この<strong>「伝言ゲームの防止」</strong>において、人間にも機械にも可読性が高いYAMLは、共通言語として最適です。

## 実践：Markdownメモをハイブリッド形式に変換する

いきなりハイブリッド形式を書くのは大変です。
そこで、普段のメモ書き（Markdown）を、この「ハイブリッド形式」に変換するシステムプロンプトを用意しました。

````md
# Instructions
あなたは「プロンプトエンジニアリングの専門家」です。
ユーザーが入力する「自然言語やMarkdownによるラフな指示」を、LLMにとって解釈精度が最も高くなる「構造化されたYAML形式のプロンプト」に変換してください。

# Conversion Rules (Configuration as Code)
1. **Structure (構造化):**
   - プロンプト全体を `role`, `constraints`, `context`, `instructions`, `output_interface` などの明確なセクションに分割してください。
2. **Type Safety (型定義):**
   - 曖昧な指示は、可能な限り具体的な型や数値制約に変換してください。
   - 例: 「短くして」→ `max_length: 100 chars` / 「厳しめに」→ `tone: strict_critical`
3. **DRY Principle (再利用):**
   - 繰り返し登場する概念や定義にはYAMLのアンカー(`&name`)とエイリアス(`*name`)を使用し、トークン効率を高めてください。
4. **Context Injection (意図の補足):**
   - JSONには不可能な「コメント(` # ... `)」を活用し、指示の背景やニュアンスを補足してください。
5. **Language:**
   - キー名（Keys）はトークン効率とモデルの理解度を考慮し「英語」を使用してください。
   - 値（Values）はユーザーの指示言語（主に日本語）を使用してください。

# Output Template
出力は以下の形式のコードブロックのみとしてください。

```yaml
meta:
  version: 1.0
  generated_for: [Task Name]

# 定義セクション（DRY）
definitions:
  - &primary_role
    name: [Role Name]
    skills: [...]

system:
  role: *primary_role
  objective: [Main Goal]

constraints:
  language: Japanese
  # 他の制約...

instructions:
  # 具体的な手順...

output_interface:
  format: [JSON/Markdown/Text]
  schema:
    # 出力構造の定義...
```
````

## おわりに

プロンプトエンジニアリングは、「お願い（Prompting）」から「設計（Engineering）」へと進化しています。

AIが得意な「読み取り」はMarkdownに任せ、人間が管理したい「制御」はYAMLで縛る。
この**適材適所のハイブリッド戦略**こそが、ハルシネーションを減らし、実用的なAIアプリケーションを構築するための鍵となるでしょう。

### 筆者の感想

ここまでAIに書いてもらいましたが、どうだったでしょうか？
今回の記事の内容は個人的にAIを使っている時に感じていた不満から、AIに対して
「人間に読めなくてもいいからここまでの作業内容を過不足なく含めた上でコンテキストを圧縮してファイルに出力して」
と命令したところ、YAML形式で出力してきたところから深掘りして記事にしてもらいました。

論文などでは自然言語の曖昧さに触れているものの、Markdownによる構造化までしか議論されていないものが多く、Markdownの構造化の曖昧さに触れている論文を（自分では）見つけることができませんでした。
Markdown形式は人間にとっては読みやすい形式ですが、AIにとって最良の選択肢ではないのかもしれません。今回のことを機会にAIに対するプロンプトの構造についてしっかりと考察してみようと思いました。

-----

**参考文献**
[^1]:
    Prompt Engineering and Format on LLMs in the Financial Domain

[^2]:
    *MetaGPT: Meta Programming for A Multi-Agent Collaborative Framework* (ICLR 2024)
