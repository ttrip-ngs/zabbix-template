# 概要
このPJは、Zabbix用のテンプレート等を管理するリポジトリです。

## 方針
- 以下のリポジトリのフォルダ構成を参考に、ディレクトリ構成を実施
 https://github.com/zabbix/community-templates
- 対応Zabbixバージョンは7.0LTS
- テンプレートはXML形式で実施
- 各テンプレート作成ごとにブランチを切って作成。ユーザが手動でZabbix上で試験完了後にmainへマージ

## Zabbix 7.0 テンプレート作成時の注意事項

### XML構造の基本

1. **dateタグは不要**
   - Zabbix 7.0では`<date>`タグは含めない
   - `<version>7.0</version>`のみで良い

```xml
<?xml version="1.0" encoding="UTF-8"?>
<zabbix_export>
  <version>7.0</version>
  <!-- <date>タグは不要 -->
  <template_groups>
```

### データ型とパラメータ

2. **value_typeは必ずFLOATを使用**
   - `UINT64`は存在しない
   - 整数値であっても`FLOAT`を使用する
   - 0/1のブール値も`FLOAT`で扱う

```xml
<value_type>FLOAT</value_type>  <!-- 正しい -->
<value_type>UINT64</value_type> <!-- エラー -->
```

3. **historyとtrendsは秒数で指定**
   - `30m`、`1d`、`365d`等の表記は使用不可
   - 必ず秒数で指定する
   - historyの最小値は3600秒（1時間）
   - 範囲: 0（無効）または 3600〜788400000秒

```xml
<history>3600</history>    <!-- 1時間（最小値） -->
<history>86400</history>   <!-- 1日 -->
<history>2592000</history> <!-- 30日 -->
<trends>31536000</trends>  <!-- 365日 -->
```

換算表:
- 1時間 = 3600秒
- 1日 = 86400秒
- 30日 = 2592000秒
- 365日 = 31536000秒

### UUID生成

4. **UUIDは必ずuuidgenで生成**
   - 適当な値や連番は不可
   - 32文字のハイフンなし形式
   - UUIDv4の仕様に準拠（13文字目が`4`、17文字目が`8/9/a/b`）

```bash
# 正しいUUID生成方法
uuidgen | tr -d '-' | tr 'A-Z' 'a-z'
```

```xml
<!-- 正しい例（uuidgenで生成） -->
<uuid>3c5265fa577a4a0595415cae2ec77e2b</uuid>

<!-- 間違った例（適当な値） -->
<uuid>a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6</uuid>
```

5. **UUIDが必要な要素**
   - template_group
   - template
   - item
   - discovery_rule
   - item_prototype
   - trigger_prototype

### アイテムタイプ

6. **Zabbix Agentのアイテムタイプ**
   - `ZABBIX_AGENT`は存在しない
   - Active checksの場合は`ZABBIX_ACTIVE`を使用
   - Passive checksの場合は`ZABBIX_PASSIVE`を使用

```xml
<type>ZABBIX_ACTIVE</type>   <!-- 正しい -->
<type>ZABBIX_AGENT</type>    <!-- エラー -->
```

### 前処理ステップ

7. **preprocessingのparameters構造**
   - `<parameters>`タグ内に`<parameter>`を配置
   - 単一パラメータでも`<parameter>`タグで囲む

```xml
<preprocessing>
  <step>
    <type>JSONPATH</type>
    <parameters>
      <parameter>$.smart_status.passed</parameter>
    </parameters>
  </step>
</preprocessing>
```

### エラーハンドリング

8. **エラーハンドラの設定**
   - JSONPATHで値が取得できない場合の対処
   - `CUSTOM_VALUE`: デフォルト値を設定
   - `CUSTOM_ERROR`: エラーメッセージを設定
   - `DISCARD_VALUE`: 値を破棄

```xml
<step>
  <type>JSONPATH</type>
  <parameters>
    <parameter>$.temperature.current</parameter>
  </parameters>
  <error_handler>CUSTOM_VALUE</error_handler>
  <error_handler_params>0</error_handler_params>
</step>
```

### テンプレート作成のベストプラクティス

9. **開発フロー**
   1. featureブランチを作成
   2. テンプレートXMLを作成
   3. Zabbixにインポートして動作確認
   4. エラーが出た場合は本ドキュメントを参照して修正
   5. 動作確認完了後、mainブランチへPR作成

10. **よくあるインポートエラーと対処法**

| エラーメッセージ | 原因 | 対処法 |
|----------------|------|--------|
| タグ"date"が間違っています | dateタグが含まれている | dateタグを削除 |
| 定数 "UINT64" が間違っています | value_typeにUINT64を使用 | FLOATに変更 |
| 値は0, 3600-788400000にしてください | historyの値が範囲外 | 3600以上の秒数に変更 |
| 32文字の長さにしてください | UUIDにハイフンが含まれている | ハイフンを削除 |
| UUIDv4で指定してください | UUIDが適当な値 | uuidgenで生成し直す |

### デバッグ方法

11. **インポートエラーの確認手順**
    1. エラーメッセージの`/1/uuid`等のパスを確認
    2. 該当するXML要素を特定
    3. 本ドキュメントの該当項目を確認
    4. 修正後、再度インポート

### 参考リンク

- [Zabbix 7.0 Documentation](https://www.zabbix.com/documentation/7.0/)
- [Zabbix Community Templates](https://github.com/zabbix/community-templates)
- [UUIDv4 Specification](https://datatracker.ietf.org/doc/html/rfc4122)
