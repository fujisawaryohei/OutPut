# Promise

## Promiseとは
非同期関数の呼び出し時に例外処理を統一化するために ES2015 から提供されているビルドインオブジェクト。

## Promiseが内部的に保持している３つの状態について
Promise オブジェクトは内部的に３つの状態を持つようになっている。  
1.Pending, 2.Fulfilled 3.Rejected

Pending は、Promise オブジェクトが生成されたタイミングの初期状態。  
resolve()を呼び出すと Fulfilled、reject()を呼び出すと rejected となる。  
これらの状態は一度セットされるとその後変更されることはないので、resolve を２度呼び出されたとしても、  
最初に resolve へ渡された値が Promise オブジェクトへラップされる。  
つまり、Promise の状態は一度状態が変更されるとそれ以降は不変となる。

```js
const promise = new Promise((resolve, reject) => {
  setTimeout(() => {
    // この時点で状態がfulFilledとなり、それ以降状態は不変であるため・・
    resolve('good!!');
    // このrejectedは無視される。
    reject(new Error('Error!'))
  }, 5000);
})

```

## Promiseの使い方
コンストラクタに resolve, reject を引数として持ったコールバック関数を引数に渡す。

```js
const execurator = (resovle, reject) => {}
new Promise(excurator)
```

Promise オブジェクトを非同期関数内で Return すると、呼び出し側において、処理の統一化を可能にすることが Promise のメリット。
```js
function dummyFetch(path) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      if(path.startsWith('/success')){
        resolve({ body: "success" })
      } 
      if(path.startsWith('/error')){
        reject(new Error("Error!"))
      }
    }, 1000);
  })
}

dummyFetch('/success').then(res => {
    console.log(res)
}).catch(err => {
    console.log(err.message)
})
```

## Promise#thenメソッド
非同期関数で Promise オブジェクトを Return することで Promise オブジェクトが持つ then メソッドを呼び出すことができる。
then は第一引数に onFulfilled コールバック関数と, 第二引数に onRejected コールバック関数を受け取る。

```js
dummyFetch('/success').then(function onFulfilled(response) {
  console.log(response)
}, function onRejected(error){
  console.log(error.message)
})
```

また下記のようにどちらも引数の省略が可能。

```js
dummyFetch('/success').then((response) => {
    console.log(response)
}, (error) => {
    console.log(error)
})
```

また onFulfilled と onRejected の２つの引数は、必要に応じて、コールバックを呼び出さないことも可能。

```js
dummyFetch('/success').then((response) => {
  console.log(response)
})
```

気をつけるべきは、下記のような例。  
dummyFetch が Rejected 状態の Promise を返しているにも関わらず、onRejected コールバックでハンドリングしていないためエラーが生じる。

```js
dummyFetch('/error').then((response) => {
  console.log(response)
})
// UnhandledPromiseRejectionWarning
```

上記のように Promise が Rejected 状態の Promise をハンドリングしたい場合。
onFulFilled コールバック部分を undefined として、第二引数の onRejeted コールバックでハンドリングする。

```js
dummyFetch('/error').then(undefined, (error) => {
    console.log(error.message)
})
```

## Promise#catchメソッド
Rejected 状態の Promise オブジェクトのみをハンドリングする際には、第一引数で undefined を渡す必要がある。
しかしそれだと分かりづらいため、catch メソッドを使ってあげると呼び出し側が明確になる。

```js
dummyFetch('/error').catch((error) => {
  console.log(error.message)
})
```

## Promiseチェイン
これまでは１つの Promise オブジェクトに対して、then や catch メソッドを呼び出して、１組のコールバック処理を
登録する（ハンドリング）するだけだった。しかし、１つの非同期処理が終わったら次の非同期処理といったように
複数の非同期処理を順番に扱いたい場合がある。Promise は、それを容易に実現するための仕組みが用意されている。

この仕組の重要なキーが、then や catch メソッドが新たな Promise インスタンスを返すという点。
このようにすることで 1 つの非同期処理のハンドリングを行った後に、再度新たな非同期処理を開始して、ハンドリングする事が可能になる。

```js
Promise.resolve(1)
// thenメソッドは新しい`Promise`インスタンスを返すので、thenメソッドを呼び出すことができる。
.then(() => {
    console.log(1);
})
.then(() => {
    console.log(2);
});
```

## Promiseチェインで値を返す
では、２つ目の非同期処理を３つ目の非同期処理に伝搬したいというケースはどうすべきか。
その場合は、then のコールバックで伝搬したい値を return すると良い。
この時の値は数値、文字列、オブジェクトなどの任意の値を渡すことができる。
```js
Promise.resolve(1)
.then((value) => {
  // 1
  console.log(value)
  return 2
}).then(value => {
  // 2
  console.log(value)
})
```

## Async Function
Async をアノテーションした関数は任意の値を Return すると、Promise オブジェクトにラップして Return する。
```js
async function doAsync() {
  return '値';
}

doAsync.then((value) => {
  console.log(value)
})
```

この時の Promise の内部状態。
- 通常通り Return した場合は、FulFilled の Promise を返す。
- Promise を Return した場合は、Promise をそのまま返す。
- 例外が発生した場合は、rejected の Promise を返す。

```js
// 1. resolveFnは値を返している
// 何もreturnしていない場合はundefinedを返したのと同じ扱いとなる
async function resolveFn() {
  return "返り値";
}
resolveFn().then(value => {
  console.log(value); // => "返り値"
});

// 2. rejectFnはPromiseインスタンスを返している
async function rejectFn() {
  return Promise.reject(new Error("エラーメッセージ"));
}

// rejectFnはRejectedなPromiseを返すのでcatchできる
rejectFn().catch(error => {
  console.log(error.message); // => "エラーメッセージ"
});

// 3. exceptionFnは例外を投げている
async function exceptionFn() {
  throw new Error("例外が発生しました");
  // 例外が発生したため、この行は実行されません
}

// Async Functionで例外が発生するとRejectedなPromiseが返される
exceptionFn().catch(error => {
  console.log(error.message); // => "例外が発生しました"
});
```

## await
await も Promise をラップして返す。
await 式では、右辺の Promise インスタンスの状態が pending から fulFilled もしくは Rejected になるまで、
その場で非同期処理の完了を待つ。そして Promise インスタンスの状態が変わると処理が再開される。

下記の例では、 axiox が Return する Promise オブジェクトの状態が変更されるまで処理がストップする。
そして、Promise オブジェクトの状態が変わったタイミングで response に Promise オブジェクトが代入されてそれが Return される。

```js
async function asyncMain() {
  const resonse = await axios.get('/hogefuga')
  return response
}
```

つまり、このような Promise を返さないビルドインの非同期関数を aysnc function で呼び出す場合は、await 時に Promise を返すようにしてあげないといけない。

```js
async function dummyFetch(path) {
  const response = await new Promise((resolve, reject) => {
    setTimeout(() => {
      if (path.startsWith('/success')){
        resolve({ body: 'success!!'})
      }
      if (path.startsWith('/error')){
        reject(new Error('Error!'))
      }
    }, 1000)
  })
  return response
}

dummyFetch('/success').then((value) => {
  console.log(value)
})
```