# 学習メモ: 例外・入力処理・継承 (Rental_System3 / 図書館の貸出システム)

DVDレンタルシステムと図書館の貸出システムを作りながら理解したことのまとめです。
「ダメなパターン」と「良いパターン」を、実際に自分のコードで起きたことと合わせて書いています。

---

## 1. 例外は「離れた場所」に任せるために使う

例外は「自分では決められないことを、呼んだ側に任せる」仕組みです。

### ダメなパターン: 同じメソッドの中で投げて、すぐ受ける

```java
void display() {
    try {
        if (!sc.hasNextInt()) {
            throw new InputMismatchException("数値を入力してください");  // 投げて
        }
        ...
    } catch (InputMismatchException e) {                              // すぐ受ける
        System.out.println(e.getMessage());
    }
}
```

投げる場所と受ける場所の間に、他のメソッドの呼び出しがありません。
これなら例外にせず、`if` と `continue` で足ります。

```java
if (!sc.hasNextInt()) {
    sc.next();
    System.out.println("数値を入力してください");
    continue;
}
```

### 良いパターン: 別のメソッドで投げて、呼んだ側で受ける

```java
// DVD クラス: 問題が起きたことは知っているが、画面にどう出すかは決められない
public void rented(LocalDate today) throws RentalException {
    if (!rented) {
        throw new RentalException("その作品は、借りられているので無理です");
    }
    ...
}

// RentalMenu クラス: どう見せるかは画面が決める
try {
    dOne.rented(today);
} catch (RentalException e) {
    System.out.println(e.getMessage());
}
```

見分け方: `try` の中でメソッドを呼んでいて、その呼び先で `throw` されているか。
クラスが同じでも別でも構いません。大事なのは「投げるメソッド」と「受けるメソッド」が別なことです。

---

## 2. 例外が出ると、そのメソッドの残りは飛ばされる

例外が投げられると、残りの処理は実行されず、対応する `catch` まで一気に戻ります。
便利ですが、**例外より前に実行した処理は取り消されません**。

### ダメなパターン

```java
manager.cal(dOne.getType(), guest.getResister());  // 売上を先に加算
dOne.rented(today);                                // ここで例外が出ても加算は残る
```

### 良いパターン: 失敗するかもしれない処理を先、状態を変える処理を後

```java
dOne.rented(today);                                // 先に貸出 (失敗ならここで例外)
manager.cal(dOne.getType(), guest.getResister());  // 成功してから課金
dvdSave.add(dOne);
```

---

## 3. キーボード入力の落とし穴

### ダメなパターン

- `nextInt()` / `next()` と `nextLine()` を混ぜる。
  `nextInt()` は改行を残すので、次の `nextLine()` が空文字を読んでしまう。
- `hasNextInt()` が false のとき、悪い入力を捨てない。同じ入力を何度も読んで無限ループになる。
- `if (sc.hasNextInt() || sc.next().length() >= 20)` のように、条件の中で `next()` を呼ぶ。
  `hasNextInt()` が false だと右側が実行され、名前が読み捨てられて、もう一度入力を待つ。

### 良いパターン: 入力は全部 `nextLine()` で読み、`parseInt` を `try/catch` で囲む

```java
DVD dOne;
try {
    int n = Integer.parseInt(sc.nextLine().trim()) - 1;
    dOne = dvd[n];                       // 範囲外は ArrayIndexOutOfBoundsException
} catch (NumberFormatException e) {
    System.out.println("半角の数字を入力してください");
    return;
} catch (ArrayIndexOutOfBoundsException e) {
    System.out.println("1から" + dvd.length + "の番号を入力してください");
    return;
}
```

`Integer.parseInt` は Java 側のコードが例外を投げるので、同じメソッド内で `catch` して構いません。

---

## 4. 継承

継承は「共通部分を親に置き、違いだけを子に書く」ための仕組みです。

- 判断の目安は「A は B の一種か」(is-a)。
  - 先生は会員の一種 → 継承してよい。
  - 社員は店長の一種ではない → 継承しない。
- 「持っている」関係 (has-a) はフィールドで持つ。
  - `Library` は本を持つ。`Shop` は社員を持つ。

### 図書館の例

```java
public abstract class Member {
    public void borrow() { ... getMaxBooks() ... }   // 共通の動き
    protected abstract int getMaxBooks();            // 上限は子が決める
}

public class Teacher extends Member {
    @Override protected int getMaxBooks() { return 5; }
}
public class Student extends Member {
    @Override protected int getMaxBooks() { return 3; }
}
```

- `borrow()` の中の `getMaxBooks()` は `this.getMaxBooks()` の省略形。
- `this` は実体 (`Teacher` か `Student`) なので、子が上書きした方が呼ばれる。
- 呼べるメソッドは参照型 (`Member`) で決まり、動く中身は実体で決まる。

---

## 5. enum は「名前付きの値」だけでなく「ルール」も持たせると強い

### 今の形

```java
public enum DVDPrice { NEW(150), NORMAL(100), OLD(50); ... }
```

マジックナンバーは消えましたが、割引ルールは `Shop.cal()` の `if` に残っています。
新しい区分を足すと、enum、`DVD` の文字列変換、`Shop.cal()` の 3 か所を直すことになります。

### 次の形 (目標)

割引率も enum に持たせれば、区分を足す変更が 1 行で済みます。
また、計算だけを返すメソッドに分ければ、画面なしでテストできます。

```java
int calcPrice(DVDPrice type, boolean register) { ... return money; }  // 計算だけ
// 表示と売上の加算は、呼ぶ側で行う
```

---

## 6. ダメ / 良いの一覧

| テーマ | ダメ | 良い |
|---|---|---|
| 例外 | 同じメソッドで投げてすぐ受ける | 別のメソッドで投げ、呼んだ側で受ける |
| 処理の順番 | お金・状態を変えてから失敗しうる処理 | 失敗しうる処理が先、状態変更は成功後 |
| 入力 | `nextInt()` と `nextLine()` の混在 | `nextLine()` + `parseInt` + `try/catch` |
| 入力 | 悪い入力を捨てずにループ | 毎回 1 行読んで捨てる |
| 継承 | 社員が店長を継承 (is-a でない) | 共通の親 (`Employee`) から `Manager` が継承 |
| 計算 | 計算・表示・売上加算が 1 メソッド | 計算は値を返すだけ、表示は呼ぶ側 |
| 命名 | `NORAML`、`OlD` | `NORMAL`、`OLD` (enum 定数は大文字) |

---

## 7. 次にやること

- 料金計算を画面から切り出し、JUnit でテストを書く。
- データをファイルに保存し、再起動しても残るようにする。
- パッケージ名を小文字にする。`System.out` をドメインのクラスから外す。
