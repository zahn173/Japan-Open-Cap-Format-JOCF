# JSON Schema to OpenAPI - 実装手順

## 実装時のルール
- 各手順ごとに、t_wadaさんのTDDに則って実装する
- テストコードは `utils/json-schema-to-openapi/tests` に実装する
- 実装内容は[spec.md](spec.md)を参照すること

## 手順
[ ] `functions.to_yaml_field_with_indent`を実装 > `utils/json-schema-to-openapi/src/to_yaml_field_with_indent.py`
[ ] `functions.field_to_yaml_string`を実装 > `utils/json-schema-to-openapi/src/field_to_yaml_string.py`
[ ] `functions.json_schema_to_yaml_string`を実装 > `utils/json-schema-to-openapi/src/json_schema_to_yaml_string.py`
- メイン処理を `utils/json-schema-to-openapi/src/json_schema_to_open_api.py` に実装 
    [ ] Step1を実装
    [ ] Step2を実装
    [ ] Step3を実装
    [ ] Step4を実装
    [ ] Step5を実装
    [ ] Step6を実装
    [ ] Step7を実装
    [ ] Step8を実装
