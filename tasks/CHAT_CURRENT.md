# CHAT_CURRENT

更新日: 2026-10-14

## プロジェクト

新販売事例

## GitHub

`xxxinakenxxx-hash/New-Sales-Cases`

2026-10-14、ユーザーから正式な保存先として指定された。

リポジトリの存在、default branchが `main` であること、ChatGPT接続にpush権限があることを確認した。

## 現在地

旧「機械販売事例まとめアプリ」を継ぎ足して修正する方針を終了し、新プロジェクトとして正本から再設計を開始した。

GitHubを正本管理の中心とし、正本文書をMarkdownで管理する。

案件概要書・要件定義書・統合設計書・API・内部関数一覧の4正本を作成し、4文書のクロスレビューと矛盾修正まで完了した段階。

## 確定事項

- プロジェクト名: 新販売事例
- GitHubリポジトリ: `xxxinakenxxx-hash/New-Sales-Cases`
- 正本管理: GitHub
- 正本文書: Markdown
- 開発体制: ChatGPT → Claude Code
- ChatGPT: 設計、正本更新、Claude Code報告の検査、次指示、完了判定
- Claude Code: 実装、テスト、実装結果記録
- ユーザー: 目的・仕様の承認、実機確認、MVP完了確定
- `tasks/CHAT_CURRENT.md`: ChatGPT側の判断・現在地
- `tasks/TASK_CURRENT.md`: Claude Code側の実装進捗・結果
- 両進捗ファイルは正本ではない
- 「なぜやり直すのか」を別記録として残す
- 旧プロジェクトの正本・工程表・指示書を新正本として継承しない
- 現行コードの再利用判定は新正本完成後に行う

## 初期作成ファイル

- `docs/記録/なぜやり直すのか.md`
- `docs/記録/開発運用ルール.md`
- `docs/正本/案件概要書.md`
- `tasks/CHAT_CURRENT.md`
- `tasks/TASK_CURRENT.md`
- `README.md`
- `CLAUDE.md`

## 旧プロジェクトから残す知見

- 旧Gemの「材料全体を理解し、不足だけを1回にまとめて聞く」方式
- 旧ネタ帳10項目
- 営業担当者の言葉・温度感を残す
- 課題 → 壁 → 突破 → 成果 → 喜びのストーリー
- candidate / confirmed の分離
- fact根拠追跡
- confirmed-only編纂
- 履歴・version管理
- 構造化検索

## 旧プロジェクトで確認した失敗

- 逐次1問方式が長期化した
- semantic重複除外後にfallback質問へ偏った
- `AI_QUESTION_DECISION_INVALID` に至った
- 会話上の返答までcandidate化した
- candidateが細分化・重複し、因果関係が弱くなった
- 技術的な成功と販売事例としての完成品質が乖離した

## 現在までに作成した正本

- `docs/正本/案件概要書.md`
- `docs/正本/要件定義書.md`
- `docs/正本/統合設計書.md`
- `docs/正本/API・内部関数一覧.md`

## 追加確定事項

- 承認フローはMVPでは使用しない。
- 将来使えるよう承認の枠だけ設計上残す。
- 通常フローでは編纂確認後、そのまま社内公開する。

## MVP範囲の追加確定

- 資料添付はMVP必須
- A4印刷・PDF出力はMVP必須
- 匿名共有版はMVP対象外
- 承認フローはMVPでは使わず、将来用の枠だけ残す
- 旧データの初期移行は行わない
- ランキング機能はMVP対象外
- 最低限の利用ログ・分析ログは記録する
- 社内の権限ある利用者には確認済みの顧客名・金額を通常表示できる
- 社外公開はMVPでは行わない

## 正本クロスレビュー結果

- 実施日: 2026-10-14
- 対象: 案件概要書 / 要件定義書 / 統合設計書 / API・内部関数一覧
- 承認フローの旧記載を除去
- 資料添付必須の表現を統一
- A4/PDFはMVPで1事例単位、複数一括出力は対象外と確定
- 検索条件の不足を統合設計・APIへ反映
- 手動fact追加はcandidate開始へ統一
- suspended状態を定義
- 判定: 開発工程表・テスト計画の作成へ進行可
- 記録: `docs/記録/正本クロスレビュー_2026-10-14.md`

## 第2クロスレビュー修正

ユーザーによる全文照合報告13件を独立確認し、以下を修正した。

- target_process: 現在の統合設計にも反映済みであることを確認
- 編纂構造へ result を追加
- publicationへ visibility / showCustomerName / showAmounts を追加
- CaseCardDTOへcustomerNameを追加
- 「事実確認へ戻る」API・状態遷移を追加
- 公開済み事例の改訂フローを追加
- unpublish後はcompilation_reviewへ戻す
- suspendedの停止・再開APIを追加
- draft→ready_for_aiをmarkCaseReadyForAIで明示
- later質問の再回答フローを追加
- manager用listManagedCasesを追加
- 途中離脱を派生指標として定義
- requestAdditionalInterviewへexpectedVersion追加
- 質問0件レスポンスをquestions: []へ統一
- confirmed参照資料の削除はATTACHMENT_IN_USEで拒否へ統一
- API§36の箇条書きを修正

## 次に行うこと

1. `docs/正本/開発工程表・テスト計画.md` を新規作成
2. 工程表確定後、工程別実装指示書を作成
3. 新正本一式確定後、旧コードを再利用／改修／廃止／新規へ分類

## 禁止事項

- 旧プロジェクトの実装を理由に新正本を曲げない
- 未確認情報を確定事項として書かない
- 旧工程番号を自動継承しない
- 旧API・旧関数・旧DB構造を先に固定しない
- Claude Codeの報告をそのまま完了判定しない
