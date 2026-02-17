# Kotlin oneOf/anyOf + kotlinx_serialization 対応状況

## 結論

| | oneOf | anyOf |
|---|---|---|
| discriminator あり | **対応済み** (既存) | — |
| discriminator なし + kotlinx_serialization | **今回修正** | **未対応** |

`oneOf` は sealed interface + value class パターンで正しく生成できるようになった。
`anyOf` は `anyof_class.mustache` に `{{#kotlinx_serialization}}` 分岐が**そもそも存在しない**ため、
kotlinx_serialization を指定しても Gson の `CustomTypeAdapterFactory` コードが出力される。

---

## oneOf: 修正内容

### 問題

`oneof_class.mustache` の kotlinx セクションは `discriminator.mappedModels` のみに依存しており、
discriminator なし oneOf（例: `oneOf: [integer, string]`）で空の sealed interface + 壊れた serializer が出力されていた。

### 修正方針

`{{#discriminator}}`/`{{^discriminator}}` で分岐を追加:

- **discriminator あり**: 既存の sealed interface + 外部 Serializer（変更なし）
- **discriminator なし**: sealed interface + `@JvmInline value class` + 外部 KSerializer（新規）

### 変更ファイル

| ファイル | 変更内容 |
|---------|---------|
| `KotlinClientCodegen.java` | `ToOneOfValueClassName` ラムダ追加（`kotlin.Long` → `LongValue`） |
| `oneof_class.mustache` | import / annotation / body に `{{^discriminator}}` 分岐追加 |
| `KotlinClientCodegenModelTest.java` | ユニットテスト追加 |
| `oneOf-primitive-types.yaml` | テスト用スペック |

### 生成結果 (oneOf: [integer(int64), string])

```kotlin
@Serializable(with = CompanyIdSerializer::class)
sealed interface CompanyId {
    @JvmInline
    value class LongValue(val value: kotlin.Long) : CompanyId

    @JvmInline
    value class StringValue(val value: kotlin.String) : CompanyId
}

object CompanyIdSerializer : KSerializer<CompanyId> {
    override val descriptor: SerialDescriptor = buildClassSerialDescriptor("CompanyId")

    override fun serialize(encoder: Encoder, value: CompanyId) {
        val jsonEncoder = encoder as? JsonEncoder ?: throw SerializationException(...)
        when (value) {
            is CompanyId.LongValue -> jsonEncoder.encodeLong(value.value)
            is CompanyId.StringValue -> jsonEncoder.encodeString(value.value)
        }
    }

    override fun deserialize(decoder: Decoder): CompanyId {
        val jsonDecoder = decoder as? JsonDecoder ?: throw SerializationException(...)
        val jsonElement = jsonDecoder.decodeJsonElement()
        val errorMessages = mutableListOf<String>()

        try {
            return CompanyId.LongValue(jsonDecoder.json.decodeFromJsonElement<kotlin.Long>(jsonElement))
        } catch (e: Exception) { errorMessages.add("...") }

        try {
            return CompanyId.StringValue(jsonDecoder.json.decodeFromJsonElement<kotlin.String>(jsonElement))
        } catch (e: Exception) { errorMessages.add("...") }

        throw SerializationException("Cannot deserialize CompanyId. ...")
    }
}
```

`./gradlew compileKotlin` でコンパイル成功確認済み。

### 仕組み: `fnToOneOfValueClassName` ラムダ

value class 名を `dataType` から生成するための Mustache ラムダを追加した。

```
入力 (dataType)    → 出力 (value class 名)
"kotlin.Long"      → "LongValue"
"kotlin.String"    → "StringValue"
"SomeModel"        → "SomeModelValue"
```

ロジック: 最後の `.` 以降を取り出して `Value` を付与。

テンプレートでの使い方:
```mustache
{{#fnToOneOfValueClassName}}{{{dataType}}}{{/fnToOneOfValueClassName}}
```

### 注意: `decodeFromJsonElement` の import

`decodeFromJsonElement<T>()` は Kotlin の **reified inline 拡張関数**。
`import kotlinx.serialization.json.decodeFromJsonElement` がないと
非 reified 版 `decodeFromJsonElement(deserializer, element)` として解決され、
引数の型が合わずコンパイルエラーになる。
実際に `./gradlew compileKotlin` で発見した問題。

---

## anyOf: 未対応の詳細

### 原因

`anyof_class.mustache` の構造を `oneof_class.mustache` と比較すると:

```
oneof_class.mustache (修正後)
├── {{#kotlinx_serialization}}
│   ├── {{#discriminator}} → sealed interface + discriminator ベース Serializer
│   └── {{^discriminator}} → sealed interface + value class + try-catch Serializer  ★ 今回追加
└── {{^kotlinx_serialization}}
    └── data class + Gson CustomTypeAdapterFactory

anyof_class.mustache (現状)
└── data class + Gson CustomTypeAdapterFactory のみ
    ※ {{#kotlinx_serialization}} 分岐が一切ない
```

### 症状

kotlinx_serialization を指定しても以下のコードが出力される:

```kotlin
// Profile1CompanyId.kt — anyOf なのに Gson コードが出る
@Serializable  // ← plain。カスタム serializer の指定なし

data class Profile1CompanyId(var actualInstance: Any? = null) {
    class CustomTypeAdapterFactory : TypeAdapterFactory {  // ← Gson 専用クラス
        override fun <T> create(gson: Gson, ...) { ... }
    }
    companion object {
        fun validateJsonElement(jsonElement: JsonElement?) { ... }  // ← Gson の JsonElement
    }
}
```

問題点:
1. `Gson`, `TypeAdapter`, `TypeAdapterFactory` 等の Gson クラスを参照 → kotlinx_serialization 環境では未依存でコンパイルエラー
2. `@Serializable` が plain なのでカスタム serializer が機能しない
3. `import java.io.IOException` が常に出力される（`{{^kotlinx_serialization}}` ガードなし）

### 修正方針（未着手）

`oneof_class.mustache` で行った修正と同じパターンを `anyof_class.mustache` にも適用する:

1. import セクションに `{{#kotlinx_serialization}}` / `{{^kotlinx_serialization}}` 分岐を追加
2. annotation に `@Serializable(with = ...)` を追加
3. body に sealed interface + value class + KSerializer パターンを追加
4. 既存 Gson コードは `{{^kotlinx_serialization}}` で囲む

oneOf と anyOf の唯一の意味的違いは「1つだけマッチ vs 複数マッチ可」だが、
kotlinx_serialization の deserialize 実装では try-catch で最初にマッチした型を返す方式なので、
テンプレートの構造はほぼ同一になる。

---

## テンプレート全体像

```
model.mustache (L:13)
├── {{#oneOf}}  → oneof_class.mustache
├── {{#anyOf}}  → anyof_class.mustache
└── それ以外     → data_class.mustache

oneof_class.mustache (修正済み)
├── import
│   ├── Gson / Moshi / Jackson (変更なし)
│   └── kotlinx_serialization
│       ├── 共通: KSerializer, SerializationException, ...
│       ├── {{#discriminator}}: JsonObject, jsonObject, jsonPrimitive
│       └── {{^discriminator}}: decodeFromJsonElement
├── annotation
│   ├── {{^generateOneOfAnyOfWrappers}}: @Serializable
│   └── {{#generateOneOfAnyOfWrappers}}: @Serializable(with = Serializer::class)
├── {{#kotlinx_serialization}} body
│   ├── {{#discriminator}}: sealed interface + mappedModels + 外部 Serializer
│   └── {{^discriminator}}: sealed interface + value class + 外部 KSerializer  ★
└── {{^kotlinx_serialization}} body
    └── data class + Gson CustomTypeAdapterFactory

anyof_class.mustache (未修正)
└── data class + Gson CustomTypeAdapterFactory のみ  ← kotlinx 分岐なし
```

## 関連ファイル

| ファイル | 役割 |
|---------|------|
| `modules/.../resources/kotlin-client/oneof_class.mustache` | oneOf テンプレート（**修正済み**） |
| `modules/.../resources/kotlin-client/anyof_class.mustache` | anyOf テンプレート（**未修正**） |
| `modules/.../resources/kotlin-client/model.mustache` L:13 | テンプレート分岐起点 |
| `modules/.../languages/KotlinClientCodegen.java` | `fnToOneOfValueClassName` ラムダ等 |
| `modules/.../resources/kotlin-client/libraries/multiplatform/oneof_class.mustache` | 参考実装 |
