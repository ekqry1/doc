# C++20 の std::ranges::views 逆引きマニュアル

C++20 から導入された Ranges は、STL コンテナに対する操作を直観的に記述できる。
この記事では、Ranges の特に view によって何ができるかを見る。
（view とは何かについては[この記事](https://zenn.dev/onihusube/articles/6608a0185832dc51213c)などを参考のこと）

以下のコードは全て次のテンプレートに従う。

```c++
#include <iostream>
#include <ranges>
#include <unordered_map>
#include <vector>

int main()
{
    namespace vw = std::views;
    std::vector test0{1, 2, 3, 4, 5};
    /* ここにコード */
    return 0;
}
```

コンパイラは GCC 10 を使用した。外部ライブラリは特に必要ない。

## vector を view に、view を vector に変換する

vector を view に変換するには`vw::all`を用いる。
view から vector への変換は、直接の方法は提供されていないので、
view の`begin`と`end`を vector のコンストラクタに渡して変換する。

```c++
    auto vw1 = vw::all(test0);                 // viewに変換
    std::vector vec1(vw1.begin(), vw1.end());  // vectorに変換
```

## 連番の view を生成する

`vw::iota`が使える。生成したい範囲が$[a, b)$のとき`vw::iota(a, b)`として呼び出す
（`b`は出力に含まれない）。

```c++
    auto vw6 = vw::iota(1, 6);
    std::vector vec6(vw6.begin(), vw6.end());  // == {1, 2, 3, 4, 5}
```

$b$を省略すると無限列が得られる。この場合、vector に変換しようとすると
コンパイルエラーになる。

```c++
    auto vw7 = vw::iota(1);
    // std::vector vec7(vw7.begin(), vw7.end());  // コンパイルエラー！
```

## view の各要素を変換する

`std::views::transform`を使う。例えば`test0`の要素を 2 倍するようなコードは次のように書ける。

```c++
    auto vw2 = vw::all(test0) |                             // viewに変換
               vw::transform([](int i) { return i * 2; });  // 各要素を2倍
    std::vector vec2(vw2.begin(), vw2.end());               // == {2, 4, 6, 8, 10}
```

view は元の vector への参照を持つため、元々のベクタの中身を書き換えると出力も変わる。

```c++
    test0[0] = 10;                             // 元の配列を書き換え
    std::vector vec3(vw2.begin(), vw2.end());  // == {20, 4, 6, 8, 10}
```

ちなみに、この例のように vector に一度だけ操作を適用するような場合は、
view を経由せずとも次のようにシンプルに書ける。

```c++
    std::vector vec2 = vw::transform(test0, [](int i) { return i * 2; });
```

以下で紹介する操作についても同様である。
view に対する操作を`|`でつなぐ上のような書き方は、
操作を連結して行いたい場合に便利である。

## view の要素をフィルタリングする

`vw::filter`が使える。例えば偶数だけを取り出すようなコードは
次の通り。

```c++
    auto vw4 = vw::all(test0) |                               // viewに変換
               vw::filter([](int i) { return i % 2 == 0; });  // 偶数のみ選択
    std::vector vec4(vw4.begin(), vw4.end());                 // == {2, 4}
```

## view の要素を反転させる

`vw::reverse`が使える。

```c++
    auto vw5 = vw::all(test0) |                // viewに変換
               vw::reverse;                    // 反転
    std::vector vec5(vw5.begin(), vw5.end());  // == {5, 4, 3, 2, 1}
```

## view を平坦化（flatten）する

`vw::join`が使える。

```c++
    std::vector<std::vector<int>> test1{{1}, {2, 3}, {4, 5, 6}};
    auto vw8 = vw::all(test1) |                // viewに変換
               vw::join;                       // 平坦化（flatten）
    std::vector vec8(vw8.begin(), vw8.end());  // == {1, 2, 3, 4, 5, 6}
```

`vector<string_view>`に対して用いると文字列の連結が可能である。

```c++
    using namespace std::literals;
    std::vector test2 = {"foo"sv, "-"sv, "bar"sv, "-"sv, "baz"sv};
    auto vw9 = vw::all(test2) | vw::join;
    std::string str9(vw9.begin(), vw9.end());  // == "foo-bar-baz"
```

## view を分割する

`vw::split`が使える。例えば`test0`を`3`で分割すると次のようになる。

```c++
    auto vw10 = vw::all(test0) |  // viewに変換
                vw::split(3) |    // 3で分割
                vw::transform([](auto v) {
                    // 一発でvectorには変換できないのでとりあえず内部のviewを変換する
                    auto c = vw::common(v);  // begin/endで得る型を一致させる
                    return std::vector<int>(c.begin(), c.end());
                });
```

`string_view`に対して用いると文字列の分割ができる。

```c++
    std::string_view words = "foo; bar; baz", delim = "; ";
    auto vw11 = words |                     // 既にviewなので変換不要
                vw::split(delim) |          // delimによって分割
                vw::transform([](auto v) {  // 結果をstringに変換
                    auto c = vw::common(v);
                    return std::string(c.begin(), c.end());
                });
    std::vector vec11(vw11.begin(), vw11.end());  // == {"foo", "bar", "baz"}
```

## view の先頭を取り出す

`vw::take(n)`を使うと先頭の`n`個を取り出せる。`n`が view のサイズより大きい場合は
単純に view 全体が取得できる。

```c++
    auto vw12 = vw::all(test0) |                  // viewに変換
                vw::take(3);                      // 先頭3個を取る
    std::vector vec12(vw12.begin(), vw12.end());  // == {1, 2, 3}

    auto vw13 =
        vw::iota(100) |  // 100から連番の無限列
        vw::take(3) |    // 先頭3個を取る
        vw::common;  // vectorに変換するためにbegin/endで得る型を一致させる
    std::vector vec13(vw13.begin(), vw13.end());  // == {100, 101, 102}
```

`vw::take_while`を使うと条件に合致する要素を取得できる。

```c++
    auto vw14 = vw::iota(100) |             // 100から連番の無限列
                vw::take_while([](int v) {  // 103より小さい間先頭から取る
                    return v < 103;
                }) |
                vw::common;  // begin/endの型を一致させる
    std::vector vec14(vw14.begin(), vw14.end());  // == {100, 101, 102}
```

## view の先頭を削除する

`vw::drop_while`を使うと、条件に合致する限り先頭から要素を削除できる。

```c++
    auto vw15 =
        vw::iota(1, 6) |                              // {1,2,3,4,5}を生成
        vw::drop_while([](int i) { return i < 3; });  // 先頭から<3の要素を削除
    std::vector vec15(vw15.begin(), vw15.end());  // {3, 4, 5}
```

## view に入ったペア・タプルの要素を選択する

`vw::element`を使う。ペアの場合は`vw::keys`と`vw::values`が使える。
view は`map`や`unordered_map`でもよい。

```c++
    std::vector<std::tuple<int, int, int>> vec = {
        {1, 2, 3}, {4, 5, 6}, {7, 8, 9}};
    auto vw16 = vw::all(vec) |    // viewに変換
                vw::elements<1>;  // タプルの1番目の要素を選択
    std::vector vec16(vw16.begin(), vw16.end());  // {2, 5, 8}

    std::unordered_map<std::string, int> map = {
        {"key1", 1},
        {"key2", 2},
        {"key3", 3},
    };
    auto vw17 = map | vw::keys;                   // mapのkeyを取得
    std::vector vec17(vw17.begin(), vw17.end());  // {"key3", "key2", "key1"}
    auto vw18 = map | vw::values;                 // mapのvalueを取得
    std::vector vec18(vw18.begin(), vw18.end());  // {3, 2, 1}
```
