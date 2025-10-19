# JSON Schema → OpenAPI 3.1 変換実装 - 方針

## 目的

JOCFの既存JSON Schemaから、各言語の型クラスを自動生成可能なOpenAPI 3.1ドキュメントを生成する。

## 背景

### 現状の課題
- JOCFはJSON Schemaで型定義を管理している
- 各言語の型クラス（TypeScript、Python等）を手動で保守するのはコストが高い
- 型の一貫性を保つのが困難

### 解決策
OpenAPI 3.1を経由することで、既存のコード生成ツール（OpenAPI Generator等）を活用し、多言語対応を実現する。

## 技術選定

### OpenAPI 3.1を選択した理由
- **JSON Schema 2020-12と100%互換**: 変換コストが最小
- **成熟したエコシステム**: 20+言語のコード生成ツールが存在
- **豊富な周辺ツール**: ドキュメント生成、バリデーション、モックサーバー等

### $refの解決 -> `utils/json-validator/validator/schema_loader.py` を再利用する
- **$ref-$id解決**: json-validatorで実装済み
- **実装コスト最小**: 実装コストが最小
