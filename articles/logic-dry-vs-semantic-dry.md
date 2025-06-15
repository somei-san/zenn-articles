---
title: "処理のDRYか？意味のDRYか？" # 記事のタイトル
emoji: "⚔️" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["DRY"] # タグ。["markdown", "rust", "aws"]のように指定する
published: true # 公開設定（falseにすると下書き）
published_at: 2023-01-08 # Qiita の記事が元なのでそちらの公開日に合わせる
---

## DRYの弁別

[DRY原則](https://ja.wikipedia.org/wiki/Don't_repeat_yours)と呼ばれるものがある。雑に表現すると、同じ内容を複数場所で表現すると後でお前が死ぬぞ、ということだ。
プログラマなら箸の持ち方より先に学ぶ大原則であるが、脳死で適応し続けるとプロダクトが死ぬパターンも存在する。
記述をまとめすぎた事によって、可読性が下がったり変更に弱くなったり経験がある人も多いと思う。
そういったリスクを防ぐために __「処理のDRY」__ と __「意味のDRY」__ を分けて考えることを提案したい。

例えば以下のようなコードがあるとしよう

```:python
applePrice = 100
grapePrice = 200

# 休日料金
appleHolidayFee = applePrice * 1.1
grapeHolidayFee = grapePrice * 1.1

# 税込み料金
appleTaxIncludedAmount = applePrice * 1.1
grapeTaxIncludedAmount = grapePrice * 1.1
```

DRY原則を愚直にやるなら以下のような形式にまとめたくなるかもしれない


```:python
applePrice = 100
grapePrice = 200

# 休日料金
appleHolidayFee = calcTenthsIncrease(applePrice)
grapeHolidayFee = calcTenthsIncrease(grapePrice)

# 税込み料金
appleTaxIncludedAmount = calcTenthsIncrease(applePrice)
grapeTaxIncludedAmount = calcTenthsIncrease(grapePrice)

def calcTenthsIncrease(basePrice):
  return basePrice * 1.1
```

しかし、休日料金と税込み料金は本質的に異なるものである。
客足が減って休日での割増率を減らしたらどうだろう？あるいは法改正で消費税率が75%になったら？...しんどい作業が迫りくるのは想像に難くない。
処理の具体的な内容だけに注目したDRY原則の適用は、将来の変更に対してコストとリスクを上げる場合があるのだ。

各々の処理の意味に注目に注目すれば以下のような形式にできる。

```:python
applePrice = 100
grapePrice = 200

# 休日料金
appleHolidayFee = calcHolidayFee(applePrice)
grapeHolidayFee = calcHolidayFee(grapePrice)

# 税込み料金
appleTaxIncludedAmount = calcTaxIncludedAmount(applePrice)
grapeTaxIncludedAmount = calcTaxIncludedAmount(grapePrice)

def calcHolidayFee(basePrice):
  return basePrice * 1.1

def calcTaxIncludedAmount(basePrice):
  return basePrice * 1.1
```

コード行は増えるがこちらのほうが変更への耐性はずっと上がることだろう

## 総括

処理のDRYは当然重要。例えば Generics のない言語は実務上やっぱしんどい。
しかしながら、とにかくDRYだ！ひたすらDRYだ！！死んでもDRYだ！！！ってテンションだと後々つらいことになるパターンも存在する。
処理のDRYと意味のDRYの区別を頭の片隅に入れておくと、未来のあたなが助かることもあるかもね🫵
