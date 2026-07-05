# About Managing Manifest Files
&nbsp;&nbsp;&nbsp; APKファイルには、1つのマニフェストファイルしか含まれないので、メインソースセット（src/main/res）、ビルドバリアント、インポートされたライブラリによって提供される複数のマニフェストファイルをマージする必要がある。

## Merge priorities
&nbsp;&nbsp;&nbsp; マージの優先順位は次の通りである。<br>

1. ビルドバリアントのマニフェストファイル<br>
ビルドバリアントに複数のマニフェストファイルがある場合、優先順位は次の通りになる。<br>
- ビルドバリアントのマニフェストファイル
- ビルドタイプのマニフェストファイル
- プロダクトフレーバーのマニフェストファイル（フレーバーディメンションでグループ化している場合、最初にリストされているフレーバーディメンションが最も優先順位が高い）
2. アプリモジュールのメインマニフェストファイル
3. インポートされたライブラリのマニフェストファイル

## Merge conflict heuristics
&nbsp;&nbsp;&nbsp; 優先度の低いマニフェストの要素が、優先度の高いマニフェストにない場合、そのままマージされたマニフェストに追加される。優先度の低いマニフェストと優先度の高いマニフェストに、同じ要素があれば結合しようとし、その要素の中で異なる値を持つ同じ属性がある場合、マージの競合が発生する。<br>
&nbsp;&nbsp;&nbsp; 一方で、以下のような例外もある。
- `<manifest>`要素内の属性は、最も優先度の高いマニフェストの属性のみが使用される。
- `<uses-feature>`と`<uses-library>`要素内の`android:required`属性は、一方のマニフェストで要求された機能やライブラリが必ず含まれる
- `<intent-filter>`要素は、すべて追加される。

## Merge rule markers
&nbsp;&nbsp;&nbsp; マージルールマーカーとは、マージの競合を解決するために使用できるXML属性である。まず、マージルールマーカーは、`tools`namespaceに属するため、以下のように`<manifest>`属性で、`tools`を宣言する必要がある。
```
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.myapp"
    xmlns:tools="http://schemas.android.com/tools">
```

### node marker
&nbsp;&nbsp;&nbsp; 特定の要素内の属性、および全ての子タグに、マージルールを適用する。以下は、その一覧である。具体的なマージのされ方は[このリンク](https://developer.android.com/build/manage-manifests#node_markers)で確認する。
- `tools:node=merge` ... 競合が発生しない場合は、全ての属性とネストされた全ての要素（子タグ）をマージする。
- `tools:node="mergeOnlyAttributes"` ... 特定の要素内の属性のみをマージする。
- `tools:node="remove"` ... マージされたマニフェストファイルから、特定の要素を削除する。
- `tools:node="removeAll"` ... マージされたマニフェストファイルから、特定の要素タイプに一致する全ての要素を削除する。
- `tools:node="replace"` ... 優先度の低いマニフェストに、一致する要素がある場合は、それを無視して、優先度の高いマニフェストの特定の要素をそのまま使用する。
- `tools:node="strict"` ... 優先度の低いマニフェストの要素と、優先度の高いマニフェストの要素が完全に一致しない場合、ビルドエラーを生成する。ただし、マージルールマーカーを追加することで、解決できる。

### attribute marker
&nbsp;&nbsp;&nbsp; 要素内の特定の属性にのみマージルールを適用するには、以下の属性を使用する。各属性は、`tools:replace="android:theme,android:exported"`のように、カンマで区切って複数の属性を指定できる。
- `tools:remove="attr, "` ... 優先度の低いマニフェストに特定の属性が含まれている場合、マージされたマニフェストから特定の属性を削除する。
- `tools:replace="attr, "` ... 優先度の低いマニフェストの特定の属性と、優先度の高いマニフェストの特定の属性で置き換える。
- `tools:strict="attr, "` ... 優先度の低いマニフェストの特定の属性が、優先度の低いマニフェストの特定の属性と一致しない場合、ビルドエラーが発生する。

&nbsp;&nbsp;&nbsp; 1つの要素に複数のマーカーを適用することもできる。

### marker selector
&nbsp;&nbsp;&nbsp; `tools:selector`にパッケージ名を指定すると、特定のライブラリにマージルールマーカーを適用できる。
```
<permission android:name="permissionOne"
    tools:node="remove"
    tools:selector="com.example.lib1">
```

&nbsp;&nbsp;&nbsp; メインのマニフェストより、`minSdk`の値が高いライブラリをインポートしようとすると、エラーが発生する。そこで、メインのマニフェストの`<uses-sdk>`タグの`overrideLibrary`属性に、該当のライブラリのパッケージ名を指定することで、メインのマニフェストの`minSdk`を適用できる。
 
### Implicit system permissions
&nbsp;&nbsp;&nbsp; 優先度の低いマニフェストの`targetSdkVersion`が低く、`targetSdkVersion`の値がシステム権限の制限が追加されたバージョン以上の場合、マージされたマニフェストファイルには、明示的に権限が追加される。

### How to find conflicts?
&nbsp;&nbsp;&nbsp; 以下は、マージされたマニフェストファイルのプレピューを確認する方法である。<br>
1. Android Studioで、`AndroidManifest.xml`ファイルを開く
2. エディター下部の「マージ済みマニフェスト」タブをクリックする

競合が発生した場合、マージエラーが表示され、競合を解決するための推奨方法が示される。

## Merge policy
&nbsp;&nbsp;&nbsp; 2つのマニフェストに同じXML属性が含まれる場合、[マニフェスト要素のマージポリシーとマッチキー](https://developer.android.com/build/manage-manifests#merge_policies)に従い、マージをしたり、子要素のみを結合（最優先のマニフェストの属性のみを保持）したり、共通の親要素にそのまま追加したりする。

## How to inject build variables into the manifest?
&nbsp;&nbsp;&nbsp; ビルドバリアントごとに、マニフェストファイルの値を変えることができる。例えば、`build.gradle`ファイルに、`free`と`pro`のapllicationIdSuffixがあったとする。
```
android {
    defaultConfig {
        applicationId "com.example.myapp"
    }
    flavorDimensions "type"
    productFlavors {
        free {
            applicationIdSuffix ".free"
            dimension "type"
        }
        pro {
            applicationIdSuffix ".pro"
            dimension "type"
        }
    }
}
```
次に、マニフェストファイルに、アプリケーションidを`${applicationId}`という形で、挿入する。
```
<intent-filter ... >
    <action android:name="${applicationId}.TRANSMOGRIFY" />
    ...
</intent-filter>
```
例えば、suffixが`free`となると次のようになる。
```
<intent-filter ... >
   <action android:name="com.example.myapp.free.TRANSMOGRIFY" />
    ...
</intent-filter>
```

## Reference
https://developer.android.com/build/manage-manifests