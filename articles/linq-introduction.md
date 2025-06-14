---
title: "LINQ 概論" # 記事のタイトル
emoji: "❓" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["csharp", "LINQ", "dotnet"] # タグ。["markdown", "rust", "aws"]のように指定する
published: true # 公開設定（falseにすると下書き）
published_at: 2022-03-25 00:00 # Qiita の記事が元なのでそちらの公開日に合わせる
---

## LINQ の仕組み

仕組みを知ると、訳も分からんままに混乱することがなくなるよね？

前提

1. 対象者｜理屈はよくわからんけど、Select や Where を使ってはいる人
2. ここでは LINQ To Object だけ対象とする
3. 継承や Interface の説明は無し
4. Object 型の詳細説明は無し
5. イテレータの説明はさかのぼると GoF デザインパターンとかまで行くので小さめにした
6. 遅延実行の説明は無し。自信がないから（とくにパフォーマンス的なところ）

## 導入

LINQ を理解するためには以下の４要素への基礎知識が必要。これがわかると、LINQ を使う分には問題がなくなる。

1. 拡張メソッド
2. デリゲート
3. ジェネリクス
4. イテレータオブジェクト

### Any()もどき Vol.0

LINQ で用いられる技術の実装サンプルとして LINQ の Any() 関数的なものを、実装していく。何の工夫もないとこのようなクソ関数ができる。

```C#
// 定義
public static class LinqLike{
   public static bool Any(List<int> source)
   {
      return source.count > 0;
   }
}

// 使う側
List<int> source = new List<int>(){1,2,3};

if(LinqLike.Any(source))
   Console.WriteLine("表示される");
```

## 拡張メソッド

既存のクラスに後付けで機能を拡張できる機能。手を加えられないクラスの機能拡張、IF そのものの機能拡張に利用できる。
たとえば、

```C#
string source = "1";
int result = Int.Parse(source);
```

と書くのが面倒なら

```C#
// staticクラスのstaticメソッドとして書く
public static class extends{
   public int static ParseToInt(this string source) // 第一引数にthisをつける
      => Int.Parse(source);
}
```

を用意して、以下のように書ける。一般ユーザが触れない `string` に、機能を追加出来たのがわかる。

```C#
string source = "1";
int result = source.ParseToInt(); // こっちの方が、オブジェクト指向感が増す
```

### Any()もどき Vol.1 拡張メソッド活用

これを用いると Any() もどきはこう書ける

```C#
// 定義
public static class LinqLike{
   public static bool Any(this List<int> source)
   {
      return source.count > 0;
   }
}

// 使う側
List<int> source = new List<int>(){1,2,3};

// Listも一般ユーザが定義を触ることが出来ないが、機能を追加できた
if(source.Any())
   Console.WriteLine("表示される");
```

## デリゲート

関数を変数のように受け渡しするための仕組み。関数ポインタ。
詳しくは、以下のページ参考。

- [参考1](https://ufcpp.net/study/csharp/sp_delegate.html)
- [参考2](https://ufcpp.net/study/csharp/misc_delegate.html)

### 1. デリゲート

以下のように、デリゲートを通して間接的に関数を呼び出すことができる。
サンプルは参照サイトの引用に改変を加えたもの。

```C#
// SomeDelegate という名前のデリゲート型を定義
delegate int SomeDelegate(int a); // 引数int、戻値intというIF定義も兼ねている

class DelegateTest
{
  static void Main()
  {
    // SomeDelegate型の変数にメソッドを代入。
    SomeDelegate a = new SomeDelegate(FuncA);

    int b = a(256);  // デリゲートを介してメソッドを呼び出す。
    Console.WriteLine(b);       // 512のはず
  }

  static int FuncA(int n)
  {
    return n*2;
  }
}
```

### 2. 標準デリゲート

いちいちデリゲートの型宣言を行うのは正直だるい。標準で用意されているデリゲート型 `Func<T, TResult>` があるのでそれを使おう。そうするとこう書ける。

```C#
class DelegateTest
{
  static void Main()
  {
    // 引数int,戻り値intの標準デリゲートにメソッドを代入
    Func<int, int> a = FuncA;

    int b = a(256);  // デリゲートを介してメソッドを呼び出す。
    Console.WriteLine(b);       // 512のはず
  }

  static int FuncA(int n)
  {
    return n*2;
  }
}
```

`Func<T, TResult>` においてジェネリック（後述）の最後に書かれている `TResult` が戻り値の型。引数を２つ渡したい場合は `Func<T1,  T2 , TResult>` を使う。この場合も戻り値の型は `TResult`。 引数最大12件まで行けるのかな？
戻り値がVoidの場合は `Action<T>` を使いましょう。

### 3. 高階関数

LINQ 的な文脈では、デリゲートだけだと何の役にも立たない（イベント呼び出しなんかでは別）。処理を関数に渡すことによって、関数内の処理を柔軟に切り替えれらると嬉しい。こういった関数を引数にとる関数のことを __高階関数__ と呼ぶ。

```C#
class DelegateTest
{
  static void Main()
  {
      // 渡す関数で処理を切り替え

      int source = 256;

     // 512のはず
     WriteConsole(source, FuncA); // Func<int, int>というIFに合致する関数なら何でも渡せる

     // 768のはず
     WriteConsole(source, FuncB); // Func<int, int>というIFに合致する関数なら何でも渡せる
  }

   // 関数を引数にとる関数
  static void WriteConsole(int source, Func<int, int> func){
      int precessed = func(source); // デリゲートを通して処理
      Console.WriteLine(precessed);
  }

  static int FuncA(int n)
  {
    return n*2;
  }

   static int FuncB(int n)
  {
    return n*3;
  }
}
```

### 4. 匿名関数

LINQ を使うような場面では、渡したいデリゲートの内容は比較的単純であることが多い。そうでない場合は、お前が悪い。単純な式をいちいち明確に定義して渡すのはメンド臭すぎる。そんな時のために、①関数名がいらない ＆ ②その場で書き捨て出来る、という便利な関数定義方法 __匿名関数__ が存在する。匿名関数の書き方には __匿名メソッド式__ と __ラムダ式__ が用意されているが、ここではラムダ式についてのみ説明する。匿名メソッド式は正直死ぬまで使わないからな。

ラムダ式を使うとデリゲート宣言を次のように書ける。

```C#
Func<int, bool> func = (int n) => { return n > 10; }
```

型が推論できる場合は、引数の型宣言もいらない。次の例では `Func<int, bool>` の宣言から推論できるので、ラムダ式内での型宣言が省けている。

```C#
Func<int, bool> func = n => { return n > 10; }
```

さらに、匿名関数の中身が `return` 文だけの場合は `return` も `{}` も省ける。

```C#
Func<int, bool> func = n =>  n > 10;
```

### Any()もどき Vol.2 デリゲート活用

これを用いると Any() もどきはこう書ける

```C#
// 定義
public static class LinqLike{
   public static bool Any(this List<int> source, Func<int, bool> predicate)
   {
      foreach(int s in source) {
         if (predicate(s)){
            return true;
         }
      }

      return false;
   }
}

// 使う側
List<int> source = new List<int>(){1,2,3};

// 評価式を渡して処理を制御できる
if(source.Any(x => x > 1))
   Console.WriteLine("表示される");

if(source.Any(x => x > 10))
   Console.WriteLine("表示されない");
```

## ジェネリクス

型に依存せずに処理を共有化するための仕組み。
詳しくは、このの[ページ](https://ufcpp.net/study/csharp/sp2_generics.html)参考。

具体的な型を指定してよければ文字列化する関数を以下のように書くことができる。

```C#
class GenericsTest
{
  static void Main()
  {
     // GetString(int source) が呼ばれる
     string iString = GetString(10);

     // GetString(double source) が呼ばれる
     string dString = GetString(10.0);

     // GetString(string source) が呼ばれる
     string sString = GetString("10");
  }

  static string GetString(int source)
      => source.ToString();

  static string GetString(double source)
      => source.ToString();

   static string GetString(string source)
      => source.ToString();
}
```

このとき、ジェネリクスを使うと以下のように型を抽象かして関数定義を共有化できる。`<T>` が ①ジェネリクスを使いますよー & ②暫定の型名を T とおきますよー、ということを示す。

```C#
class GenericsTest
{
  static void Main()
  {
     // GetString(int a)が"生成されて"呼ばれる
     string i = GetString(10);

     // GetString(double a)が"生成されて"呼ばれる
     string d = GetString(10.0);

     // GetString(string a) が"生成されて"呼ばれる
     string s = GetString("10");

     // 引数から型推論してくれるが、明示することもできる
     string i_2 = GetString<int>(10);
  }

   // ジェネリクスで型を抽象化した関数
   public static string GetString<T>(T a)
      => a.ToString();
}
```

しかし、このままだと T にどんなことが出来るのかがわからんので、渡された方も大したことができない。`ToString()` は Object 型(全ての型の基底クラス)にいる関数なので、呼べてるだけ。
そこで、 `where` を使って継承関係を明示する事により、① 渡される T に制限を掛ける & ② T の持つ機能をはっきりさせて使用可能にする、ことが出来る。

```C#
class GenericsTest
{
  static void Main()
  {
     // intはIComparableを継承しているので Max(int a, int b) が"生成されて"呼ばれる
     int i = Max(1, 2);

     // doubleはIComparableを継承しているので Max(double a, double b) が"生成されて"呼ばれる
     double d = Max(1.0, 2.0);

     // 配列はIComparableを継承していないので、コンパイルエラー
     int[] a_i = Max(new int[] {1, 2}, new int[] {3, 4});
  }

   // whereでIComparableを継承した型じゃないとダメだよ、を明示
   public static T Max<T>(T a, T b)
      where T : IComparable
      => a.CompareTo(b) > 0 ? a : b; // TはIComparableを継承しているはずなので,T.CompareTo(T)が呼べる
}
```

### Any()もどき Vol.3 ジェネリクス活用

これを用いると Any() もどきはこう書ける

```C#
// 定義
public static class LinqLike{
   public static bool Any(this List<T> source, Func<T, bool> predicate)
   {
      foreach(T s in source) {
         if (predicate(s)){
            return true;
         }
      }

      return false;;
   }
}

// 使う側 int編
List<int> source = new List<int>(){1,2,3};

if(source.Any(x => x > 1)) // bool Any(this List<int> source, Func<int, bool> predicate) を呼んでいるのと同じ
   Console.WriteLine("表示される");

// 使う側 bool編
List<bool> source2 = new List<bool>(){false, true, false};

// 異なる型のリスト、異なる型を引数にとる関数でも同じAny()でチェックできる
if(source2.Any(x => x is true)) // bool Any(this List<bool> source, Func<bool, bool> predicate) を呼んでいるのと同じ
   Console.WriteLine("表示される");
```

## イテレータオブジェクト

GoF デザインパターンの一つ。今の言語なら大体機能として組み込まれているはず。foreach で回すための仕様と考えてＯＫ。C# では `IEnumerable<T>` インターフェースが「私はイテレータオブジェクトですよ～」という意味。

詳しく言うと、イテレート可能IF `IEnumerable<T>` と私イテレータだよIF `IEnumerator<T>` は別なんだけど割愛
詳細はこれらを参照

- [イメージをつかみやすい](https://qiita.com/kerochan/items/13bbbbb14c3a309c7ec4)
- [詳しく知りたいならここ](https://ufcpp.net/study/csharp/sp_foreach.html)

`IEnumerable<T>` と各種コレクションの継承関係は次のようになっている。
※1 右が左を継承している
※2 抜粋。C#のコレクションはいっぱいあるので

|  反復可能だよIF  | コレクションだよIF | 各コレクション毎IF |     具象クラス     |
| :--------------: | :----------------: | :----------------: | :----------------: |
| `IEnumerable<T>` |  `ICollection<T>`  |     `IList<T>`     |     `List<T>`      |
|                  |         ⬑          |         ⬑          | `ImmutableList<T>` |
|                  |         ⬑          |  `IDictionary<T>`  |  `Dictionary<T>`   |
|                  |         ⬑          |     `ISet<T>`      |    `HashSet<T>`    |

上の表のように、C# におけるコレクションは全て `ICollection<T>` を継承しており、さらに `ICollection<T>` は `IEnumerable<T>` を継承している。 ← __これ重要__

以上を踏まえて LINQ の IF を見てみる。次に示すのは Where() の関数 IF。

```C#
public static IEnumerable<TSource> Where<TSource>(this IEnumerable<TSource> source, Func<TSource, bool> predicate)
```

`IEnumerable<T>` に対して処理をするようになっている！！！！！！！！
なので、LINQ は C# の全てのコレクションに対して使用できる！！！！！！！！！

### IEnumerable 補足

このことはリストに対して LINQ 使うと戻り値がリストでなくなる現象の理由でもある。初心者のころみんなもビビったよね？LINQ はあくまで `IEnumerable<T>` を貰って `IEnumerable<T>` を返すからなんだよね。
LINQ の戻り値をそのまま `foreach` できるのも、戻り値の型が `IEnumerable<T>` だからだね。

### Any()もどき Vol.4 イテレータIF活用

これを用いると Any() もどきはこう書ける
__本物の Any() と同じ挙動をするようになった!__

```C#
// 定義
public static class LinqLike{
   public static bool Any(this IEnumerable<T> source, Func<T, bool> predicate)
   {
      foreach(T s in source) {
         if (predicate(s)) {
            return true;
         }
      }

      return false;;
   }
}

// 使う側
List<int> sourceList = new List<int>(){1, 2, 3};
int[] sourceArray = {1, 2, 3};
HashSet sourceSet = new HashSet<int>(){1, 2, 3};

// 具体的なコレクションが違っても同じ方法で処理できる！
if(sourceList.Any(x => x == 1))
   Console.WriteLine("表示される");
if(sourceArray.Any(x => x == 2))
   Console.WriteLine("表示される");
if(sourceSet.Any(x => x == 3))
   Console.WriteLine("表示される");
```

## まとめ

LINQ はなれないと挙動でテンパることがあるけど、仕組みをしると愛着がわく。

## 補足

この記事は[この記事](https://qiita.com/goyaYellow/items/c2e736254be3e8fa447d)の複製です