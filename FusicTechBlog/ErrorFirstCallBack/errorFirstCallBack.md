## 非同期関数と例外処理
非同期関数の例外をハンドリングしたい時。

```js
try {
  // fetch(1000ミリ秒後に例外がスローされる)
  setTimeout(() => {
    //成功時の処理
    // ~
    //失敗時の処理
    throw Error.new("error!")
  }, 1000)
} catch(err) {
  console.log(err.message)
}
```

```js
setTimeout(() => {
  try {
    // 成功時の処理
  } catch {
    throw Error.new("error!")
  }
}, 1000);
```

ただし、提供されている npm が非同期関数の場合…↑ができない。

こういった時の実装パターンとして・・エラーファーストコールバックを使用する。
非同期関数呼び出し時のコールバックの第一引数に Error オブジェクトを渡す実装パターン。

Node.js ビルドインパッケージの fs パッケージの非同期関数である readFile 関数ではこのパターンが使用される。
```js
fs.readFile("./example.txt", (error, data) => {
  if(error){
    console.error(error)
  } else {
    console.log(data)
  }
})
```

エラ-ファーストコールバックを適用した非同期関数を実装してみる。
```js
function dummyFetch(path, callback) {
  setTimeout(() => {
    if(path.startsWith("/success")) {
      callback(null, { body: "success" })
    } else {
      callback(new Error("NOT FOUND"), null)
    }
  }, 1000);
}

dummyFetch("/success", (error, response) => { 
  if(error) {
    console.log(error.message)
  } else {
    console.log(response)
  }
})

dummyFetch("/failure", (error, response) => {
  if(error) {
    console.log(error.message)
  } else {
    console.log(response)
  }
})
```