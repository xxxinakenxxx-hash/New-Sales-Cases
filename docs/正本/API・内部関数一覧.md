# 新販売事例 API・内部関数一覧

- 文書区分: 正本
- 状態: 初版・ユーザー確認中
- 作成日: 2026-10-14
- プロジェクト名: 新販売事例
- GitHub: `xxxinakenxxx-hash/New-Sales-Cases`
- 上位文書:
  - `docs/正本/案件概要書.md`
  - `docs/正本/要件定義書.md`
  - `docs/正本/統合設計書.md`

## 1. 本書の目的

本書は、統合設計書で定義した業務責務を、実装時に利用する公開API・内部関数・DTO・エラー・呼出関係へ落とし込む。

本書は業務責務を示す論理API定義とする。物理実装は `docs/正本/技術選定理由書.md` の技術構成に従う。

HTTP endpointやサーバーモジュールへ落とし込む際も、ここで定める責務境界は維持する。

## 2. 命名規則

### 2.1 公開API

画面またはクライアントから呼び出す業務APIは lowerCamelCase とする。

例:

- `createCase()`
- `saveCaseInput()`
- `runCaseUnderstanding()`

### 2.2 内部関数

サーバー内部だけで使用する関数は末尾に `_` を付ける。

例:

- `validateCaseAccess_()`
- `buildUnderstandingContext_()`

### 2.3 DTO

入出力の論理構造は PascalCase + `DTO` とする。

例:

- `CaseDTO`
- `FactDTO`
- `CompilationDTO`

## 3. 共通ルール

すべての更新系APIは、必要な場合に以下を受け取る。

- `caseId`
- `expectedVersion`
- 実行者コンテキスト

重要更新では、サーバー側のversionと一致しない場合に上書きせず `CONFLICT` を返す。

作成、AI実行、アップロード完了、公開、コメント投稿、PDF生成等の再送が起こりうる処理は、requestIdまたはidempotency keyで同一要求の二重実行を防ぐ。

すべての公開APIはサーバー側で認証・認可する。

AI生成結果を自動でconfirmedへ進めない。

## 4. 共通レスポンス

### ApiResultDTO

```text
success
data
error
requestId
currentState
```

### ApiErrorDTO

```text
code
userMessage
retryable
details
requestId
currentState
```

`details` に秘密値、AI API応答全文、内部stack、認証情報を含めない。

## 5. 共通エラーコード

最低限、以下を定義する。

- `AUTH_REQUIRED`
- `FORBIDDEN`
- `NOT_FOUND`
- `VALIDATION_ERROR`
- `CONFLICT`
- `INVALID_STATE`
- `AI_ERROR`
- `AI_OUTPUT_INVALID`
- `ATTACHMENT_ERROR`
- `ATTACHMENT_UNREADABLE`
- `ATTACHMENT_IN_USE`
- `FACT_SOURCE_INVALID`
- `FACT_CONFLICT`
- `COMPILATION_NOT_READY`
- `COMPILATION_SOURCE_INVALID`
- `PUBLICATION_FAILED`
- `SEARCH_FAILED`
- `PRINT_FAILED`
- `INTERNAL_ERROR`

## 6. 認証・利用者コンテキスト

### getCurrentUser()

公開API。

入力: なし

出力: `UserContextDTO`

責務:

- 現在の利用者を取得
- role
- branch
- userId
- active
- permissions
  を返す

### validateCurrentUser_()

内部関数。

責務:

- 認証済みか
- activeか
- 必要role・permissionを持つか
  を検証する

### resolveUserPermissions_()

内部関数。

責務:

- role
- branch
- 案件ownership
- 管轄範囲
  から実効権限を決定する

## 7. Case API

### createCase()

公開API。

入力: `CreateCaseInputDTO`

出力: `CaseDTO`

責務:

- 新規caseを作成
- 初期状態 `draft`
- owner
- branch
- createdBy
- version
  を設定
- `case_created` を記録

### saveCaseInput(caseId, input, expectedVersion)

公開API。

入力: `CaseInputDTO`

出力: `CaseSaveResultDTO`

責務:

- S-02入力を保存
- 既存sourceを上書きせずversion管理
- worksheet / memo sourceを更新
- 中断再開可能な状態を保持
- 保存後もAI処理を自動実行しない

### getCase(caseId)

公開API。

出力: `CaseDetailForEditDTO`

責務:

- 案件基本情報
- 入力済みsource
- attachment状態
- current state
- version
  を返す

### getCaseResumeState(caseId)

公開API。

出力: `ResumeStateDTO`

責務:

- 現在の案件状態
- 次に開く画面
- 未完了処理
- 保存済み状態
  を返す

### listMyCases(criteria)

公開API。

出力: `CaseListDTO`

責務:

- 自分の作成途中
- 要確認
- 編纂確認待ち
- 公開済み
  を一覧する

### markCaseReadyForAI(caseId, expectedVersion)

公開API。

責務:

- S-02の「AIで整理する」操作
- AIへ渡せるsourceが1件以上あることを検証
- `draft → ready_for_ai` へ遷移する

### suspendCase(caseId, expectedVersion)

公開API。

責務:

- 対象案件を `suspended` へ遷移
- 停止前状態を保持
- `case_suspended` を記録

### resumeCase(caseId, expectedVersion)

公開API。

責務:

- `suspended_from_state` に基づき停止前状態へ戻す
- `case_resumed` を記録

### startCaseRevision(caseId, expectedVersion)

公開API。

責務:

- 公開済み事例の改訂を開始
- `case_revision_started` を記録
- caseを `fact_review` へ戻す
- 現在のpublicationは維持し、旧公開版を検索・閲覧可能なままにする
- 新しいcompilationを公開するまで公開版を差し替えない

### reopenFactReview(caseId, compilationId, reason, expectedVersion)

公開API。

責務:

- S-05の「事実確認へ戻る」
- `fact_review_reopened` を記録
- `compilation_review → fact_review` へ遷移
- 現在のcompilationは履歴として保持

### archiveCase(caseId, expectedVersion)

公開API。

責務:

- 案件をarchivedへ
- 物理削除しない
- 公開中なら公開停止と整合させる

### restoreCase(caseId, expectedVersion)

公開API。

責務:

- archivedから復元
- 元状態または安全な再開状態へ戻す

## 8. Case内部関数

### validateCaseAccess_(caseId, action, userContext)

責務:

- owner
- role
- action
- publication status
  に基づき操作可否を判定
- 公開済み事例の検索・閲覧ではbranchを認可条件に使わない

### normalizeCaseInput_(input)

責務:

- 入力値のtrim
- 空値正規化
- 型検査
- 不正値除外

### persistCaseSource_(caseId, sourceType, content, actor)

責務:

- source版を保存
- source_id発行
- 旧版履歴保持

### transitionCaseState_(caseId, from, to, expectedVersion)

責務:

- 許可された状態遷移だけを実行
- 不正遷移は `INVALID_STATE`

## 9. Attachment API

資料添付はMVP必須。

### createAttachmentUpload(caseId, metadata)

公開API。

出力: `AttachmentUploadDTO`

責務:

- アップロード開始情報を発行
- case権限を確認
- 許可形式・サイズ等を検証

`docs/正本/技術選定理由書.md` に従い、Google Cloud Storageへの期限付きアップロードURLまたは同等の安全なアップロード情報を発行する。内部storage keyをクライアントへ直接公開しない。

### completeAttachmentUpload(caseId, attachmentId, expectedVersion)

公開API。

出力: `AttachmentDTO`

責務:

- アップロード完了確認
- metadata保存
- extraction待ちへ遷移

### removeAttachment(caseId, attachmentId, expectedVersion)

公開API。

責務:

- attachmentを論理削除
- そのattachmentだけを根拠にする未confirmed candidateへの影響を記録
- confirmed factが参照している場合は `ATTACHMENT_IN_USE` として削除を拒否する

### getAttachments(caseId)

公開API。

出力: `AttachmentDTO[]`

### getAttachmentDownloadUrl(caseId, attachmentId)

公開API。

出力: `AttachmentDownloadDTO`

責務:

- case権限を確認
- 対象attachmentが有効であることを確認
- 短時間だけ有効なダウンロードURLまたは同等の取得情報を返す
- 内部storage keyを返さない

### retryAttachmentExtraction(caseId, attachmentId, expectedVersion)

公開API。

責務:

- 読取失敗した資料を再抽出

## 10. Attachment内部関数

### validateAttachmentMetadata_(metadata)

責務:

- MIME
- 拡張子
- サイズ
- 件数
- 禁止形式
  を検証

### storeAttachmentBinary_(...)

責務:

- 実ファイル保存

物理ストレージは `docs/正本/技術選定理由書.md` に従いGoogle Cloud Storageを使用する。内部storage keyを通常のclient DTOへ露出しない。

### extractAttachmentText_(attachment)

責務:

- PDF
- 画像OCR
- 文書
- 表計算
- テキスト
  から利用可能なテキストを抽出

### persistAttachmentExtraction_(attachmentId, result)

責務:

- extraction status
- extracted text
- evidence ranges
- extraction error
  を保存

### buildAttachmentSource_(attachment)

責務:

- AI理解に渡すsource形式へ変換

## 11. AI案件理解 API

### runCaseUnderstanding(caseId, expectedVersion)

公開API。

出力: `CaseUnderstandingResultDTO`

責務:

- 案件全体を一度に評価
- candidate fact抽出
- 重複正規化
- 不足評価
- 質問バッチ生成
- 質問0件を正常系として扱う
- 自動confirmed化しない

### getCaseUnderstanding(caseId)

公開API。

出力: `CaseUnderstandingDTO`

責務:

- AIが把握した要点
- candidate生成状況
- 不足評価
- question batch有無
  を返す

### requestAdditionalInterview(caseId, reason, expectedVersion)

公開API。

責務:

- 利用者が明示的に追加取材を要求
- 例外的な2回目以降の質問生成を許可

## 12. AI案件理解 内部関数

### collectCaseSources_(caseId)

責務:

- worksheet
- memo
- interview answers
- attachment
- manual facts
  を収集

### buildUnderstandingContext_(caseId, sources, facts, answers)

責務:

- AI入力文脈を生成
- 無関係他案件を含めない
- 秘密値・内部ログを除外

### extractFactCandidates_(context)

責務:

- candidate fact候補を抽出
- 根拠source_id・evidence rangeを要求

### normalizeFactCandidates_(candidates)

責務:

- domain正規化
- 表記統一
- 明らかな会話制御文の除外
- 過剰細分化抑制

### detectCandidateDuplicates_(caseId, candidates)

責務:

- 完全一致
- 同義
- 同domain近似
- confirmedとの重複
  を検出

### detectFactConflicts_(caseId, candidates)

責務:

- 数値
- 日付
- 固有名詞
- 結果
  等の矛盾を検出

### evaluateCaseQuality_(caseId, context, facts)

出力: `CaseQualityDTO`

評価軸:

- 課題
- 提案
- 選定理由
- 壁
- 突破
- 営業判断
- 結果
- 顧客の言葉
- 再利用性
- 因果関係

### rankMissingInformation_(quality, context)

出力: `MissingInformationDTO[]`

責務:

- 空欄数ではなく販売価値で不足を優先評価

### generateQuestionBatch_(missingInformation, context)

出力: `QuestionBatchDraftDTO`

責務:

- 原則1回分の質問バッチを生成
- 固定件数上限を前提にしない

### validateQuestionBatch_(batch, context)

責務:

以下を除外。

- 既知情報
- 回答済み情報
- 同義質問
- 低価値質問
- 抽象質問
- 誘導質問
- 推測前提質問

全質問除外でもエラーにしない。

### canGenerateSecondBatch_(caseId, reason)

責務:

2回目以降を許す条件を検証。

許可条件:

- 新たな重大矛盾
- 新たな重要論点
- 利用者の明示的な追加取材要求

## 13. Interview API

### getQuestionBatch(caseId)

公開API。

出力: `QuestionBatchDTO`

質問0件の場合もDTOを返し、`questions: []`、`status: "not_required"` とする。

### submitQuestionBatch(caseId, batchId, answers, expectedVersion)

公開API。

出力: `InterviewSubmitResultDTO`

責務:

- 一括回答保存
- answered / unknown / later / skipped保存
- 回答をsource化
- candidate fact更新
- 自動で次質問を生成しない
- 原則fact_reviewへ進む

### updateAnswer(caseId, answerId, input, expectedVersion)

公開API。

責務:

- 回答修正
- `later → answered / unknown / skipped` の変更
- `deferred_question_answered` を記録
- source版更新
- 関連candidate再評価

### getDeferredQuestions(caseId)

公開API。

出力: `QuestionDTO[]`

責務:

- disposition=`later` の保留質問だけを返す
- fact_review以降でも再回答できるようにする

## 14. Interview内部関数

### validateQuestionBatchOwnership_(caseId, batchId)

### normalizeAnswerDisposition_(answer)

### persistInterviewAnswer_(answer)

### classifyNonFactResponse_(answer)

責務:

以下のような会話制御返答を販売事実化しない。

- 分かりません
- 意味が分かりません
- はい
- そう書きました
- 後で確認します
- その質問の意味が分かりません

### applyAnswersToCandidates_(caseId, answerIds)

責務:

- 回答を根拠とするcandidate factを更新
- 既存candidateとの重複を抑止

## 15. Fact Review API

### getFactReview(caseId)

公開API。

出力: `FactReviewDTO`

含む:

- candidate
- confirmed
- rejected
- needs_check
- evidence
- conflicts
- duplicate candidates

### confirmFact(caseId, factId, expectedVersion)

公開API。

責務:

- 根拠検証
- candidate → confirmed

### confirmFactWithEdit(caseId, factId, edit, expectedVersion)

公開API。

責務:

- 修正版factを作成
- 元factを履歴保持
- 新factをconfirmed

### rejectFact(caseId, factId, reason, expectedVersion)

公開API。

### markFactNeedsCheck(caseId, factId, note, expectedVersion)

公開API。

### addManualFact(caseId, input, expectedVersion)

公開API。

責務:

- 利用者手動fact追加
- manual source作成
- 根拠説明を保持
- 新規追加時はcandidateとして保存し、confirm操作を経てconfirmedにする

### mergeFactCandidates(caseId, factIds, mergedValue, expectedVersion)

公開API。

責務:

- 同義candidate統合
- 元candidateを履歴保持

## 16. Fact内部関数

### validateFactSource_(fact)

責務:

- source存在
- case一致
- evidence存在
  を検証

### validateFactDomain_(domain)

### supersedeFact_(oldFactId, newFactId)

### persistFactVersion_(fact)

### findDuplicateFacts_(caseId, fact)

### findConflictingFacts_(caseId, fact)

### canConfirmFact_(fact, userContext)

AI実行者だけではconfirmed不可。

## 17. Compilation API

### evaluateCompilationReadiness(caseId)

公開API。

出力: `CompilationReadinessDTO`

責務:

- confirmed factsだけで販売事例編纂が可能か評価
- 未confirmedを補完しない
- 不足があっても必ずブロックするのではなく、何が欠けるか明示

### compileCase(caseId, expectedVersion)

公開API。

出力: `CompilationDTO`

責務:

- confirmed factsだけで新しい編纂版を生成
- used fact refs保持
- 創作検査
- version発行

### getCompilation(caseId, compilationId?)

公開API。

出力: `CompilationDTO`

### saveCompilationExpressionEdits(caseId, compilationId, edits, expectedVersion)

公開API。

責務:

- 表現修正のみ許可
- 新事実混入を検知した場合は拒否
- 新事実はfact reviewへ戻す

### recompileCase(caseId, expectedVersion)

公開API。

責務:

- 新versionを作成
- 過去版を保持

### completeCompilationReview(caseId, compilationId, expectedVersion)

公開API。

責務:

- 利用者の編纂確認完了を記録
- MVPでは承認を挟まず公開可能状態へ

## 18. Compilation内部関数

### collectConfirmedFacts_(caseId)

### buildCompilationPayload_(facts)

構造:

- title
- customer_background
- problem
- proposal
- selection_reason
- alternatives
- barrier
- breakthrough
- salesperson_judgment
- collaboration
- before_after
- result
- customer_quote
- salesperson_joy
- success_factor
- caution
- target_customer
- reuse_point
- story

### generateCompilation_(payload)

### validateCompilationFactRefs_(result, facts)

### validateCompilationNoFabrication_(result, facts)

検査:

- 架空数値
- 架空顧客発言
- 架空固有名詞
- 根拠のない感情
- 未confirmed参照
- 提案と採用理由の混同
- 因果飛躍

### persistCompilationVersion_(caseId, result)

### detectNewFactsInExpressionEdit_(edits, existingCompilation)

## 19. Publication API

MVPでは承認フローを使用しない。

### publishCase(caseId, compilationId, expectedVersion)

公開API。

責務:

- 編纂確認済みか検証
- MVPでは `visibility=internal`、`showCustomerName=true`、`showAmounts=true` を適用する
- 一般利用者向けの表示設定変更APIはMVPでは設けない
- 検索索引生成
- publication更新
- publishedへ状態遷移

### unpublishCase(caseId, reason, expectedVersion)

公開API。

責務:

- 公開停止
- 検索対象から除外
- compilationは保持
- 改訂中の `fact_review` / `compilation_review` は現在状態を維持する
- case.status=`published` で改訂作業を開始していない場合のみ `compilation_review` へ戻す

### getPublicationState(caseId)

公開API。

出力: `PublicationDTO`

## 20. Publication内部関数

### validatePublicationEligibility_(caseId, compilationId)

MVP条件:

- compilation review completed
- compilation source valid
- case権限 valid

承認済みであることは要求しない。

### buildSearchIndex_(caseId, compilation)

責務:

- confirmed factと公開対象compilationから検索索引を生成
- コメント本文を索引へ含めない
- publicationの表示設定で非表示となる情報を索引・検索結果から漏らさない

### commitPublication_(caseId, compilationId, searchIndex, expectedVersion)

責務:

- publicationとsearch indexを整合更新
- 中途半端なpublished状態を作らない

### removeSearchIndex_(caseId)

## 21. 将来用 Approval 拡張ポイント

MVPでは呼び出さない。

実装する場合は正本改訂後に有効化する。

予約API名:

- `submitForApproval()`
- `approveCompilation()`
- `rejectCompilation()`
- `getApprovalState()`

MVPの通常フローには組み込まない。

## 22. Search API

### searchCases(criteria)

公開API。

出力: `SearchResultDTO`

責務:

- 構造化条件
- 全文条件
- publication状態
  を適用
- 社内公開済み事例は、認証済み社内利用者へ全社横断で返す
- 拠点・部門・manager管轄による閲覧制限はMVPでは行わない

### getCaseCards(criteria)

公開API。

出力: `CaseCardDTO[]`

カード要素:

- title
- customer name（publication.showCustomerName=true の場合）
- customer type
- problem
- proposal
- result
- barrier / breakthrough
- target customer

### getPublishedCaseDetail(caseId)

公開API。

出力: `PublishedCaseDetailDTO`

責務:

- 公開済み事例だけ取得
- 営業利用情報を返す
- 内部candidate・AIログ等を返さない

### getSearchFilterOptions()

公開API。

責務:

- 利用可能な構造化検索条件を返す

## 23. Search内部関数

### normalizeSearchCriteria_(criteria)

### searchStructuredIndex_(criteria)

### searchFullText_(terms)

### combineSearchResults_(structured, fullText)

### applyPublicationScope_(results, userContext)

### buildCaseCardDTO_(caseId)

### buildPublishedCaseDetailDTO_(caseId)

責務:

- publicationの表示設定を適用して詳細DTOを生成
- 内部candidate・AIログ・storage key・非表示情報を返さない

ベクトル検索関数はMVPでは定義しない。

## 24. Comment API

コメントは公開済み販売事例に対する社内利用者の補足・質問・横展開アイデア等を蓄積するための機能とする。

コメントはconfirmed factではなく、AI案件理解・編纂へ自動投入しない。

### getComments(caseId)

公開API。

出力: `CommentDTO[]`

責務:

- caseに有効なpublicationが存在することを検証
- caseに紐づく有効コメントを時系列で返す
- 過去publication時に投稿されたコメントも保持して返す
- 削除済みコメント本文は通常利用者へ返さない

### createComment(caseId, body)

公開API。

責務:

- 認証済み社内利用者のコメントを登録
- case.statusではなく、有効なpublicationが存在することを検証
- 投稿時点のpublicationIdを記録
- 空本文を拒否
- `comment_created` を記録

### updateComment(commentId, body, expectedVersion)

公開API。

責務:

- 投稿者本人のコメントだけ編集可能
- version競合を検出
- `comment_updated` を記録

### deleteComment(commentId, expectedVersion)

公開API。

責務:

- 投稿者本人のコメントを論理削除
- `comment_deleted` を記録

### adminDeleteComment(commentId, reason, expectedVersion)

公開API。

対象: admin / sysadmin

責務:

- 管理上必要なコメントを論理削除
- 削除理由を監査ログへ記録

## 25. Comment内部関数

### validateCommentTarget_(caseId)

責務:

- 有効なpublicationの存在を検証
- caseが改訂中のfact_review / compilation_reviewでも、有効publicationが残っていればコメント対象として許可
- 公開停止中は通常利用者の一覧取得・新規投稿を拒否

### validateCommentAuthor_(commentId, userContext)

### persistComment_(comment)

### softDeleteComment_(commentId, actor, reason?)

### sanitizeCommentForClient_(comment)

## 26. Print / PDF API

A4印刷・PDFはMVP必須。

### getPrintModel(caseId)

公開API。

出力: `PrintModelDTO`

責務:

- A4出力に必要なViewModelを生成
- 正本データから生成
- 社内利用版を対象
- publicationの表示設定を適用する
- コメント本文を含めない

### generatePdf(caseId, options)

公開API。

MVPでは1事例単位の出力を対象とし、複数事例一括PDFは対象外。

出力: `PdfResultDTO`

責務:

- A4 PDF生成
- 出力時点のpublication / compilation versionを記録

### getPrintableHtml(caseId)

公開API。

必要な技術スタックの場合のみ実装。

責務:

- ブラウザ印刷用HTMLを生成
- HTMLを正本データにはしない

## 27. Print内部関数

### buildPrintModel_(caseId, compilation, facts)

### renderPrintSections_(model)

### validatePrintContent_(model)

検査:

- 未confirmed情報混入
- 空欄の不自然な埋め
- 架空数値
- 架空顧客発言

### renderPdf_(model, options)

PDF生成は `docs/正本/技術選定理由書.md` に従い、印刷専用HTML/CSS ViewModelをheadless Chromium系rendererでPDF化する。

## 28. Admin API

### getAdminUsers(criteria)

### saveUser(input, expectedVersion)

### deactivateUser(userId, expectedVersion)

### getMasterValues(type)

### saveMasterValue(input, expectedVersion)

### getSystemSettings()

### saveSystemSettings(input, expectedVersion)

### getAuditLogs(criteria)

### getActivityLogs(criteria)

### getErrorLogs(criteria)

### getCaseAdminDetail(caseId)

管理者向けに内部状態、fact状態、attachment extraction、publication状態等を返す。

## 29. Admin内部関数

### validateAdminPermission_(userContext, action)

### normalizeUserInput_(input)

### normalizeMasterValue_(input)

### sanitizeLogRecord_(record)

### redactSensitiveFields_(record)

## 30. Analytics API

ランキング機能はMVP対象外。

### getUsageSummary(filters)

公開API。

出力: `UsageSummaryDTO`

最低集計:

- case作成開始
- AI理解実行
- 質問発生率
- 質問0件率
- 回答完了率
- fact確認完了率
- 編纂完了率
- 公開率
- 中断
- 再開
- 検索回数
- 0件検索
- 事例閲覧
- コメント投稿
- コメント編集
- コメント削除

### getFunnel(filters)

公開API。

出力: `FunnelDTO`

## 31. Analytics内部関数

### recordActivity_(event, context)

### aggregateUsage_(events, filters)

### deriveAbandonedCases_(cases, events, settings)

責務:

以下をすべて満たす案件を「途中離脱」の派生指標として集計する。

- archivedではない
- suspendedではない
- 有効なpublicationが存在しない
- 設定された一定期間、進行イベントがない

進行イベントは統合設計書§25で定義したイベント集合を用いる。

### aggregateFunnel_(events, filters)

### recordSearchEvent_(criteria, resultCount, context)

### recordCaseView_(caseId, searchId, context)

人気ランキング、営業担当ランキング等は作らない。

## 32. Error / Audit内部関数

### createRequestId_()

### logError_(code, error, context)

### writeAuditLog_(event, actor, target, before, after)

### sanitizeErrorForClient_(error)

### isRetryableError_(error)

## 33. DTO一覧

### UserContextDTO

- userId
- email
- name
- role
- branch
- permissions
- active

### CaseDTO

- caseId
- status
- ownerId
- branch
- createdAt
- updatedAt
- version

### CreateCaseInputDTO

- branch
- initialInput（任意）
- clientRequestId

### CaseSaveResultDTO

- caseId
- status
- savedSourceIds
- version
- updatedAt

### CaseDetailForEditDTO

- case
- input
- sources
- attachments
- currentState
- version

### ResumeStateDTO

- caseId
- currentState
- nextScreen
- pendingActions
- saved
- version

### CaseListDTO

- items
- total
- page

### CaseInputDTO

- customerProfile
- problem
- beforeAfter
- proposal
- alternatives
- barrier
- breakthrough
- collaborators
- customerQuote
- salespersonJoy
- memo

### AttachmentDTO

- attachmentId
- caseId
- filename
- mimeType
- size
- downloadAvailable
- extractionStatus
- extractedAt
- errorCode
- version

内部storage key / storageReferenceはclient DTOへ含めない。

### AttachmentUploadDTO

- attachmentId
- uploadUrl
- expiresAt
- requiredHeaders
- version

### AttachmentDownloadDTO

- downloadUrl
- expiresAt

### SourceDTO

- sourceId
- caseId
- sourceType
- originalContent
- createdBy
- createdAt
- version
- active

### CaseUnderstandingResultDTO

- caseId
- status
- understanding
- createdCandidateFactIds
- questionBatch
- version
- warnings

### MissingInformationDTO

- domain
- reason
- valueForCase
- priority
- evidenceRefs

### QuestionBatchDraftDTO

- reasonSummary
- questions
- status

### CaseUnderstandingDTO

- summary
- candidateCount
- quality
- missingInformation
- questionBatch
- warnings

### CaseQualityDTO

- problemQuality
- proposalQuality
- selectionReasonQuality
- barrierQuality
- breakthroughQuality
- salespersonJudgmentQuality
- resultQuality
- customerQuoteQuality
- reuseQuality
- causalCoherence
- warnings

品質値を単純な「合格点」として自動運用しない。

### QuestionBatchDTO

- batchId
- caseId
- reasonSummary
- questions
- status
- version

### QuestionDTO

- questionId
- domain
- questionText
- purpose
- priority
- status

### AnswerDTO

- answerId
- questionId
- answerText
- disposition
- createdBy
- createdAt
- version

### InterviewSubmitResultDTO

- caseId
- batchId
- savedAnswerIds
- createdOrUpdatedCandidateFactIds
- nextState
- version
- warnings

### FactDTO

- factId
- caseId
- domain
- value
- status
- sourceRefs
- evidenceText
- createdBy
- createdAt
- updatedBy
- updatedAt
- version
- supersedesFactId
- notes

### FactReviewDTO

- facts
- conflicts
- duplicates
- summary

### CompilationReadinessDTO

- ready
- missingCriticalAreas
- warnings
- confirmedFactCount

`confirmedFactCount` は品質指標ではなく参考値。

### CompilationDTO

- compilationId
- caseId
- version
- status
- sections
- usedFactIds
- createdAt
- createdBy
- reviewCompletedAt
- reviewCompletedBy

### PublicationDTO

- publicationId
- caseId
- compilationId
- status
- visibility
- showCustomerName
- showAmounts
- publishedAt
- publishedBy
- unpublishedAt
- version

### SearchCriteriaDTO

- keyword
- customerType
- problem
- targetProcess
- machine
- maker
- proposal
- selectionReason
- result
- barrier
- successFactor
- targetCustomer
- reusePoint
- branch
- dateRange
- page

### SearchResultDTO

- searchId
- total
- items
- page

### CaseCardDTO

- caseId
- title
- customerName
- customerType
- problem
- proposal
- result
- barrierBreakthrough
- targetCustomer

`customerName` は publication.showCustomerName=false の場合は返さない。

### CommentDTO

- commentId
- caseId
- publicationId
- body
- createdBy
- createdByName
- createdAt
- updatedAt
- version

### PublishedCaseDetailDTO

- caseId
- title
- customerProfile
- sections
- compilationVersion
- publishedAt

### PrintModelDTO

- title
- profile
- problem
- proposal
- beforeAfter
- companySales
- targetCustomer
- story
- customerQuote
- timeline
- barrierBreakthrough
- collaborators
- reusePoint

### PdfResultDTO

- caseId
- publicationId
- compilationId
- fileName
- downloadUrl
- expiresAt
- generatedAt

### FunnelDTO

- filters
- started
- aiUnderstood
- interviewed
- factReviewed
- compiled
- published
- abandoned

### UsageSummaryDTO

- started
- aiUnderstood
- questionGenerated
- zeroQuestion
- interviewCompleted
- factReviewCompleted
- compilationCompleted
- published
- resumed
- abandoned
- searches
- zeroResultSearches
- views
- commentsCreated
- commentsUpdated
- commentsDeleted

## 34. domain一覧

MVPのfact domainは以下を基本とする。

- `customer_profile`
- `customer_context`
- `problem`
- `target_process`
- `impact_before`
- `proposal`
- `product`
- `maker`
- `investment`
- `alternative`
- `selection_reason`
- `barrier`
- `breakthrough`
- `salesperson_judgment`
- `salesperson_action`
- `collaboration`
- `result`
- `impact_after`
- `customer_quote`
- `salesperson_joy`
- `timeline`
- `success_factor`
- `caution`
- `target_customer`
- `reuse_point`
- `company_sales`

domainは固定質問順を作るために使わない。

## 35. 画面と主要API対応

### S-01 ホーム・案件一覧

- `getCurrentUser`
- `listMyCases`
- `searchCases`

### S-02 事例入力

- `createCase`
- `saveCaseInput`
- `markCaseReadyForAI`
- `suspendCase`
- `resumeCase`
- `getCase`
- `createAttachmentUpload`
- `completeAttachmentUpload`
- `getAttachments`
- `runCaseUnderstanding`

### S-03 AI不足確認

- `getCaseUnderstanding`
- `getQuestionBatch`
- `submitQuestionBatch`
- `requestAdditionalInterview`
- `getDeferredQuestions`

### S-04 事実確認

- `getDeferredQuestions`
- `updateAnswer`
- `getFactReview`
- `confirmFact`
- `confirmFactWithEdit`
- `rejectFact`
- `markFactNeedsCheck`
- `addManualFact`
- `mergeFactCandidates`

### S-05 編纂確認

- `evaluateCompilationReadiness`
- `compileCase`
- `getCompilation`
- `saveCompilationExpressionEdits`
- `recompileCase`
- `completeCompilationReview`
- `reopenFactReview`
- `publishCase`
- `startCaseRevision`

### S-06 承認

MVPでは未使用。

### S-07 検索・カード一覧

- `searchCases`
- `getCaseCards`
- `getSearchFilterOptions`

### S-08 事例詳細

- `getPublishedCaseDetail`
- `getComments`
- `createComment`
- `updateComment`
- `deleteComment`

### S-09 印刷・PDF

- `getPrintModel`
- `generatePdf`
- `getPrintableHtml`（必要な技術スタックの場合）

### S-10 管理

- `adminDeleteComment`
- Admin API群
- Analytics API群

## 36. 代表的な状態遷移とAPI

### 新規入力

```text
createCase
→ saveCaseInput
→ attachment upload
→ markCaseReadyForAI
→ runCaseUnderstanding
```

### 質問あり

```text
runCaseUnderstanding
→ getQuestionBatch
→ submitQuestionBatch
→ getFactReview
```

### 質問なし

```text
runCaseUnderstanding
→ getFactReview
```

### 事実確認から編纂

```text
getFactReview
→ confirm / reject / needs_check
→ evaluateCompilationReadiness
→ compileCase
→ completeCompilationReview
```

### 公開

```text
completeCompilationReview
→ publishCase
→ buildSearchIndex_
→ published
```

### 公開済み事例の改訂

```text
startCaseRevision
→ fact review
→ compileCase / recompileCase
→ completeCompilationReview
→ publishCase
```

旧publicationは再公開完了まで維持する。

### コメント

```text
getPublishedCaseDetail
→ getComments
→ createComment / updateComment / deleteComment
```

コメント内容はfactへ自動変換しない。

### 検索・再利用

```text
searchCases
→ getCaseCards
→ getPublishedCaseDetail
→ getPrintModel / generatePdf
```

## 37. 禁止するAPI設計

以下のようなAPIは作らない。

### 37.1 AI結果を即confirmedにするAPI

例:

`extractAndConfirmFacts()`

禁止。

### 37.2 1問ずつ自動連鎖するAPI

例:

`answerAndGenerateNextQuestion()`

禁止。

回答送信後に自動で次質問を生成しない。

### 37.3 fallback質問を返すだけのAPI

semantic重複でAI質問が消えた場合に、固定質問を順番に埋めるAPIを作らない。

### 37.4 HTMLを正本保存するAPI

完成HTMLを正本データとして保存しない。

### 37.5 未確認事実を編纂へ混ぜるAPI

candidate / needs_check / rejectedを確定本文根拠へ入れない。

## 38. 物理API設計で具体化する項目

技術構成は `docs/正本/技術選定理由書.md` で確定する。

以下は、その技術構成を前提に実装指示書・物理設計へ落とし込む。

- HTTP method
- endpoint path
- request body
- response status
- 認証トークン形式
- DB transaction方式
- file upload方式
- storage provider
- OCR / document parser
- AI provider / model
- PDF renderer
- 全文検索エンジン

ただし、技術選定によって本書の業務責務を変更しない。

## 39. 次工程への条件

本書確定後、`docs/正本/開発工程表・テスト計画.md` を作成する。

開発工程表では以下を基準に工程を切る。

- 各工程が独立して検査できること
- 1工程ごとに受入条件を持つこと
- UIだけ、DBだけではなく業務価値単位で切ること
- AI取材品質を技術テストだけで完了にしないこと
- A4/PDF、添付資料をMVP必須として工程へ含めること
- 承認、匿名共有、ランキング、旧データ移行をMVP工程へ入れないこと
