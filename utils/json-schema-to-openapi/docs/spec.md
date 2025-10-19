# JSON Schema to OpenAPI - 仕様

## ディレクトリ構造
JSON Schema to OpenAPIスクリプトの実装ディレクトリは `utils/json-schema-to-openapi/` とする。
- プロダクションコード: `utils/json-schema-to-openapi/src/` 
- テストコード: `utils/json-schema-to-openapi/tests/`
    - fixture: `utils/json-schema-to-openapi/tests/fixture/`

## 入出力
<parameters>
    <parameter name="output_path" description="YAMLファイルの出力先">
</parameters>
<returns/>

## JSON Schema to OpenAPIの処理の流れ
<steps>
    <step name="1.作業ファイルの作成">
        以下のOpenAPIテンプレートファイルを作業フォルダにコピー(既に存在する場合は上書き)して、作業ファイル(work_file)を作成する。
        `utils/json-schema-to-openapi/template/openapi_template.yaml`
        作業ファイルのパスは `local/tmp/json-schema-to-openapi/jocf_openapi.yaml` とする。
    </step>
    <step name="2.スキーマ準備">
        SchemaLoader(loader)の新規生成 + スキーマ読み込み `${loader}.load_all_schemas` を行なう。
        SchemaLoaderは `utils/json-validator/validator/schema_loader.py` を再利用する。
    </step>
    <step name="3.file_type_listの取得">
        ファイル種別一覧(file_type_list:List[file_type])を取得する。
        `file_type_list = ${loader}.get_file_types()`
    </step>
    <step name="4.各種変数を初期化">
        以下の変数を初期化
        - ref_schema_list:List[schema_id:string]
        - root_indent=3
    </step>
    <for each ${file_type_list} file_type>
        <step name="5-a.ファイルのスキーマを取得">
            ファイルスキーマ(file_schema)を取得
            <formula> file_schema=${loader}.get_file_schema(${file_type}) </formula>
        </step>
        <step name="5-b.ファイルスキーマをYAML形式に変換 + 状態を更新">
            ```
            // スキーマキーを取得
            schema_key = ${file_schema}.get("$id")
            // スキーマ名を文字列に追加
            yaml_string += to_yaml_field_with_indent(key=${schema_key}, indent=2)
            // ファイルスキーマをYAML文字列に変換 + 参照スキーマを取得
            (_str:schema_str,_added:added_ref_schema_list) = json_schema_to_yaml_string(schema=${file_schema}, base_indent=${root_indent}, ref_schema_list=${ref_schema_list}.copy())
            // 状態の更新
            yaml_string += ${_str}
            ref_schema_list += ${_added}
            ```
        </step>
    </for>
    <step name="6-a.未処理の参照スキーマキューを初期化">
        未処理参照スキーマキュー(waiting_ref_schema_queue:Queue[schema_id:string])を、参照スキーマ一覧で初期化
        ```
        waiting_ref_schema_queue = new Queue[${ref_schema_list}]
        ```
    </step>
    <while `${waiting_ref_schema_queue}.isEmpty==false`>
        <step name="7-a.処理対象の参照スキーマを取得">
            処理対象の参照スキーマ(ref_schema)を、スキーマローダーから取得
            ```
            // 処理対象のスキーマIDをデキュー
            ref_schema_id = ${waiting_ref_schema_queue}.deque()
            // 処理対象のスキーマを取得
            ref_schema = ${loader}.get_schema_by_id(schema_id=${ref_schema_id})
            ```
        </step>
        <step name="7-b.参照スキーマをYAML文字列に変換+状態を更新">
            ```
            // 参照スキーマのスキーマキーを取得
            schema_key = ${ref_schema}.get("$id")
            // スキーマキーをYAML文字列に追加
            yaml_string += to_yaml_field_with_indent(key=${schema_key}, indent=2)
            // 参照スキーマをYAML文字列に変換 + 参照スキーマを取得
            (_str:schema_str,_added:added_ref_schema_list) = json_schema_to_yaml_string(schema=${ref_schema}, base_indent=${root_indent}, ref_schema_list=${ref_schema_list}.copy())
            // 状態の更新
            yaml_string += ${_str}
            ref_schema_list += ${_added}
            waiting_ref_schema_queue = ${waiting_ref_schema_queue}.enqueue(${_added}) // 処理対象のキューに追加
            ```
        </step>
    </while>
    <step name="8.YAMLファイルを出力">
        ```
        // 作業ファイルに書き込み
        ${work_file}.append(${yaml_string})
        // openapi_spec_validatorでYAMLファイルを検証
        (spec_dict, spec_url) = openapi_spec_validator.readers.read_from_filename(${work_file})
        openapi_spec_validator.validate_spec(spec_dict)
        // 検証で問題なければ出力先にコピー
        move(${work_file}, ${output_path})
        ```
    </step>
</steps>
<functions>
    <function name="json_schema_to_yaml_string" description="JSONスキーマをYAML形式に変換する">
        <variants>
            <variant name="schema" type="Map[key:str,value:Any]" description="出力対象のJSONスキーマ"/>
            <variant name="base_indent" type="int" description="yamlファイルの基準となるインデント"/>
            <variant name="ref_schema_list" type="List[schema_id:string]" description="既に登録済みの参照スキーマIDの一覧">
        </variants>
        <returns>
            <return name="schema_str" type="str" description="JSONスキーマをyaml形式に変換した文字列" />
            <return name="added_ref_schema_list" type="List[schema_id:str]" description="追加された参照スキーマIDの一覧" />
        </return>
        <steps>
            <step name="JSONスキーマをyaml形式で文字列化">
                <for each ${schema.(key,value)} (key,value)>
                    <ifthen>
                        <if `${key} NOT in ("$id")`>
                            ```
                            // YAMLに不要な項目(e.g. "$id")を除いて出力
                            (_str:schema_str, _list:List[schema_id:string]) = field_to_yaml_string(key=${key}, value=${value}, base_indent=${base_indent}+1, ref_schema_list=${ref_schema_list}.copy() ∪ ${added_ref_schema_list}.copy())
                            // schema_str, ref_schema_listを更新
                            schema_str += ${_str};
                            added_ref_schema_list += ${_list};
                            ```
                        </if>
                    </ifthen>
                </for>
            </step>
        </steps>
    </function>
    <function name="field_to_yaml_string" description="JSONスキーマの属性を再帰的にYAML形式に変換する">
        <variants>
            <variant name="key" type="str" description="フィールドのキー"/>
            <variant name="value" type="Any" description="フィールドの値"/>
            <variant name="base_indent" type="int" desciption="yamlのインデント位置"/>
            <variant name="ref_schema_list" type="List[schema_id:string]" description="登録済みの参照スキーマのID一覧"/>
        </variants>
        <returns>
            <return name="schema_str" type="str" description="YAML形式の文字列"/>
            <return name="added_ref_schema_list" type="List[schema_id:string]" description="当該処理で追加された参照スキーマ"/>
        </returns>
        <steps>
            <step>
                <ifthen name="valueの型で分岐">
                    <if `${value} is instanceof Map[sub_key:str, sub_value:Any]`> 
                        <for each ${value} (sub_key, sub_value)>
                            <step name="1つ下の階層のkey-valueで再帰呼び出し">
                                ```
                                // オブジェクト・配列名を表示
                                schema_str = to_yaml_field_with_indent(key=${key}, indent=${base_indent})
                                // 再帰呼び出し
                                (_str:schema_str, _list:added_ref_schema_list) = field_to_yaml_string(key=${sub_key}, value=${sub_value},  base_indent=${base_indent}+1, ref_schema_list=${ref_schema_list}.copy());
                                // 参照スキーマリスト・追加キュー・YAML文字列を更新
                                ref_schema_list += ${_list};
                                added_ref_schema_list += ${_list};
                                schema_str += ${_str};
                                ```
                            </step>
                        </for>
                    </if>
                    <elseif `${key} is $ref`>
                        <ifthen name="参照スキーマが未登録なら登録">
                            <if `${ref_schema_list}.contains(${value})==false`>
                                ```
                                // 参照スキーマリスト・追加キューを更新
                                ${ref_schema_list} += ${value};
                                ${added_ref_schema_list} += ${value};
                                ```
                            </if>
                        </ifthen>
                        ```
                        // YAML文字列を更新
                        schema_str += to_yaml_field_with_indent(key=${key}, value=${value}, indent=${base_indent});
                        ```
                    </elseif>
                    <else> 
                        ```
                        // (key, value)を文字列化
                        schema_str += to_yaml_field_with_indent(key=${key}, value=${value}, indent=${base_indent});
                        ```
                    </else>
                </ifthen>
            </step>
        </steps>
    </function>
    <function name="to_yaml_field_with_indent" description="key-valueをインデント付きのYAML属性に変換する(末尾に改行コード付き)">
        <variants>
            <variant name="key" required=true/>
            <variant name="value" required=false default=""/>
            <variant name="indent" required=true/>
        </variants>
        <return>
            ```
            indent_str = ' ' * indent            
            return f"${indent_str}${key}: ${value}\n"            
            ```
        </return>
    </function>
</functions>