## Promise オブジェクトが登場する以前の非同期関数のエラーハンドリングのお話

非同期関数の例外処理はよくあります。
さてここで下記のような例がありますが、この書き方は正しいでしょうか。

```js
try {
  // fetch(1000ミリ秒後に例外がスローされる)
  setTimeout(() => {
    // if 成功時の処理
    // else 失敗時の処理
    throw Error.new("error!");
  }, 1000);
} catch (err) {
  console.log(err.message);
}
```

正しくありません。というのも、非同期関数は、文字通り非同期関数であるため  
この例の場合、1000 ミリ秒後でないと実際の処理は実行されません。  
try〜catch 文は同期処理であるため、try ブロック内実行時に、Error オブジェクトがスローされずに catch できなくなります。

```js
setTimeout(() => {
  try {
    // 成功時の処理
  } catch {
    throw Error.new("error!");
  }
}, 1000);
```

つまりこのように、非同期関数のエラーハンドリングを行う際には、非同期関数内でエラーハンドリングを
行う必要性があります。

しかし、非同期関数がビルドインパッケージや npm で提供されている場合、内部実装に依存するため、try ~ catch が使えません。

こういった時の実装パターンとして・・エラーファーストコールバックを使用することが ES2015 以前はありました。

そう、Promise オブジェクトが提供される前の話です。

エラーファーストコールバックとは、非同期関数呼び出し時のコールバックの第一引数に Error オブジェクトを渡す実装パターンです。  
第 2 引数には、非同期関数の処理が成功した時の値が渡されます。

その実装パターンが適応されるものとして、Node.js ビルドインパッケージの fs パッケージの非同期関数である readFile 関数があります。

```js
fs.readFile("./example.txt", (error, data) => {
  if (error) {
    console.error(error);
  } else {
    console.log(data);
  }
});
```

このように、非同期関数呼び出し時にコールバック関数を渡してますが、この時のコールバック関数の引数に注目してください。

第一引数の`error`の部分には、例えば第一引数で指定したファイルパスのファイルが存在しなかった時等にエラーが発生して、  
その時の Error オブジェクトが含まれるようになっています。

第二引数の`data`の部分には、ファイルの読み込みが成功した時の値が入ります。

面白いですね。Promise が登場する前は、このように非同期関数のエラーハンドリングを行っていたようです。  
ですが、これは 1 つデメリットが存在します。それは、ビルドインパッケージや npm を使用する開発者によってエラーハンドリング時の処理がバラバラで
統一化されないという点です。この「エラーハンドリング時の処理の統一化」を目的として、Promise オブジェクトは ES2015 からビルドインオブジェクトとして登場することになります。

Axios を自分はよく使用するのですが、なぜ Promise オブジェクトを返却するのかよく理解せずに使ってました。しかしこの話を知ってしっくり来ました。

使用している言語や、ライブラリ、パッケージの歴史を遡る事で、なぜそれが使用されているのかという部分を意識してコードが書けるので
誤った使用方法を避けることができるのではないかと感じました。

では、最後に、応用としてエラーファーストコールバックを適用した擬似的に fetch する非同期関数を実装してみます。

```js
function dummyFetch(path, callback) {
  setTimeout(() => {
    if (path.startsWith("/success")) {
      callback(null, { body: "success" });
    } else {
      callback(new Error("NOT FOUND"), null);
    }
  }, 1000);
}

dummyFetch("/success", (error, response) => {
  if (error) {
    console.log(error.message);
  } else {
    console.log(response); // { body: "success" }
  }
});

dummyFetch("/failure", (error, response) => {
  if (error) {
    console.log(error.message); // "NOT FOUND"
  } else {
    console.log(response);
  }
});
```

最後まで読んで頂きありがとうございました。
