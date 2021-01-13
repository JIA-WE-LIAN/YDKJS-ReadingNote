## Promise

### 待解決的問題

在 Callback 的章節，我們發現 Callback 的兩個問題，一個是順序性難以閱讀、一個是一個是缺乏信任度，並且有一個很重要的問題就是控制反轉，我們將寫好的 `function` 傳給第三方工具，並祈禱它們可以如我們所想的運作。

### 用生活的例子來比喻Promise

想像一個場景，我們到一家速食餐廳櫃台，點了一個起司漢堡，接著拿到一張點單號碼 ( 通常不會立即送餐給我們 )，然後我們開始用這張點單號碼去代表一個未來的起司漢堡來進行思考，好比我們可能會想找朋友來一起享用、或者想要在另外點一杯手搖杯搭配......，沒多久叫到了我們的號碼，並且拿到我們期待的起司漢堡，當然可能還有另一種情況，就是店員告訴我們起司漢堡已經販售完，我們只能抱著失望的心情離開櫃台。

在這個故事中，我們一直用未來的值去思考，我們沒特別注意漢堡什麼時候會來，而且我們相信一定會有一個明確的結果會回來，要嘛就是餐點做好了 ( fulliment )，要嘛就是餐點沒有了 ( reject )。

> 在程式中會稍微複雜點，可能不會有回應。

### 用 Callback 來比喻 Promise

看看下面這段程式碼 :

``` JavaScript
function add(getX, getY, cb) {
    var x, y;
    getX(function(xVal) {
        x = xVal;
        // both are ready?
        if (y != undefined) {
            cb(x + y); // send along sum
        }
    });
    getY(function(yVal) {
        y = yVal;
        // both are ready?
        if (x != undefined) {
            cb(x + y); // send along sum
        }
    });
}

// `fetchX()` and `fetchY()` are sync or async
// functions
add(fetchX, fetchY, function(sum) {
    console.log(sum); // that was easy, huh?
});
```

這段 Callback 程式碼展現了 Promise 的特點，當使用 `add` 方法 ( 從外部來看 )，不論是 `fetchX` 、 `fetchY` 我們都變得不是很在乎他是用同步的方式還是用非同步的方式來拿到資料，這段程式碼去除了時間的影響，這就是 Promise 的其中一項特點，說得更明白點，不論是同步還是非同步，Promise 都將其視為非同步，Promise 將時間給封裝了起來，變成得以操作所有的流程。

### Promise 的不可變性

來看一段簡短的 Promise 的程式碼 :

``` JavaScript
function add(xPromise, yPromise) {
    // `Promise.all([ .. ])` takes an array of promises,
    // and returns a new promise that waits on them
    // all to finish
    return Promise.all([xPromise, yPromise])

        // when that promise is resolved, let's take the
        // received `X` and `Y` values and add them together.
        .then(function(values) {
            // `values` is an array of the messages from the
            // previously resolved promises
            return values[0] + values[1];
        });
}

// `fetchX()` and `fetchY()` return promises for
// their respective values, which may be ready
// *now* or *later*.
add(fetchX(), fetchY())

    // we get a promise back for the sum of those
    // two numbers.
    // now we chain-call `then(..)` to wait for the
    // resolution of that returned promise.
    .then(function(sum) {
        console.log(sum); // that was easier!
    });
```

第一段程式碼， `Promise.all` 會等待兩個 `xPromise` 、 `yPromise` 都完成 ( `fulfillment` ) ，就會跳到 `then` 的區塊，執行內部的 `function` ，雖然內部的 `function` 是寫下了 `return values[0] + values[1];` ，然而如果用一個變數將其接起來，看到的會是另一個 `Promise` 物件，這個 `Promise` 將 `values[0] + values[1]` 的值給封裝了起來。

這種 `.then(..)` 會在回傳一個 `Promise` 展現了一個特點，一種鏈式結構的寫法，它讓我們可以一直用 `.then(..)` 來控制整個 `Promise` 的流程，下一個章節會在細講，另一個特點是當 `Promise.all` 解析完成後，就無法再被重新解析 ( 當它已經是 `fulfillment` 或是 `rejection` 時，就無法在轉變成原本的 `pending` ，此物件內部的值也無法在被更動 )，這讓 Promise 可以被安全的傳到任何地方，也不會因為意外或者惡意的被修改，而且可重複觀察多次 ( `resolve` )，每次都必定有相同的結果 ( 無論是同步還是非同步 )。

### 事件監聽的類比

下面這段程式碼，是一個虛構的程式碼，嘗試用監聽事件來比喻 Promise :

``` JavaScript
function foo(x) {
    // start doing something that could take a while

    // make a `listener` event notification
    // capability to return

    return listener;
}

var evt = foo(42);

evt.on("completion", function() {
    // now we can do the next step!
});

evt.on("failure", function(err) {
    // oops, something went wrong in `foo(..)`
});
```

想像 `clickEvent` 的類比，只是將這種事件替換成一種非同步請求，過去的 Callback 在呼叫資料的同時，也必須事先定義好 Callback function 的內容，而 Promise 則是可以將非同步的事情以及 Callback function 兩個分開來寫，完成了一種控制的反逆轉 ( 原本的 Callback 是一種控制反轉 )，並且更好地完成了關注點分離。

### Promise 的信任

* 過早或過晚呼叫

前面提到 Callback 的第三方工具，可能會造成 function 過早或是過晚呼叫，而 Promise 將一切都轉化成非同步行為，也就不會有這樣的問題，看看下面這段程式碼 :

``` JavaScript
p.then(function() {
    p.then(function() {
        console.log("C");
    });
    console.log("A");
});
p.then(function() {
    console.log("B");
});
// A B C
```

它不會因為突如其來的同步行為，而印出 C -> A -> B，反而會照著它該有的順序印出 A -> B -> C。

#### Promise 中的 Promise

``` JavaScript
var p3 = new Promise(function(resolve, reject) {
    resolve("B");
});

var p1 = new Promise(function(resolve, reject) {
    resolve(p3);
});

var p2 = new Promise(function(resolve, reject) {
    resolve("A");
});

p1.then(function(v) {
    console.log(v);
});

p2.then(function(v) {
    console.log(v);
});

// A B  <-- not  B A  as you might expect
```

`p1` 的 `resolve` 的結果並不是一個立即值，而是另一個 `promise p3` ，後者會被在解析成 `B` ，然而是被非同步的解析，所以 `p1` 的 callback 會在 `p2` 的 callback 之後。

#### 解決永遠無法解析完成的 Promise

``` JavaScript
// a utility for timing out a Promise
function timeoutPromise(delay) {
    return new Promise(function(resolve, reject) {
        setTimeout(function() {
            reject("Timeout!");
        }, delay);
    });
}

// setup a timeout for `foo()`
Promise.race([
        foo(), // attempt `foo()`
        timeoutPromise(3000) // give it 3 seconds
    ])
    .then(
        function() {
            // `foo(..)` fulfilled in time!
        },
        function(err) {
            // either `foo()` rejected, or it just
            // didn't finish in time, so inspect
            // `err` to know which
        }
    );
```

* 呼叫太多次或是太少次

Promise 只能夠被 `resolve` 或是 `reject` 一次，當在一個 function 中想多次 `resolve` 或是 `reject` ，只有第一次會成功，其他的會被無聲無息地吞掉。

* 沒有傳入必要的參數

Promise 一定有解析完的值，也因此在 `.then(..)` 中 `fulfillment` 跟 `rejection` 一定有一個參數，其他的參數都會被忽略掉，如果想帶入多個參數，就傳入陣列或是物件。

* 錯誤處理的問題

Promise 的 `then(..)` 中帶有兩個參數，第二個 function 確實可以處理錯誤，就像下面的例子一樣 :

``` JavaScript
var p = new Promise(function(resolve, reject) {
    foo.bar(); // `foo` is not defined, so error!
    resolve(42); // never gets here :(
});

p.then(
    function fulfilled() {
        // never gets here :(
    },
    function rejected(err) {
        // `err` will be a `TypeError` exception object
        // from the `foo.bar()` line.
    }
);
```

然而如果是在 `fulfilled` 出錯又該怎麼辦 ? 它並不會被後面的 `rejected` 給接到，因為只能夠被解析依次，如果在這種情況要做錯誤處理，可以在 `then(..)` 的後面補上 `catch(..)` 。

#### Promise.resolve(..)

如果注意到的話，會發現 Promise 並沒有真的把 Callback 給消除掉，它只是改變了決定 Callback 的時機，這對非同步的處理有很大的影響，另一個較為細微的部分，則是 `Promise.then(..)` ，如果在內部傳了非 `thenable` 、非 Promise 的物件，它會立即解析內部的值並傳出一個 Promise 物件，那內部就是一個 Promise 物件呢 ? 他會原封不動的把這個物件回傳，那什麼是 `thenable` 物件 ? 看看下面的程式碼 :

``` JavaScript
var p = {
    then: function(cb, errcb) {
        cb(42);
        errcb("evil laugh");
    }
};

p
    .then(
        function fulfilled(val) {
            console.log(val); // 42
        },
        function rejected(err) {
            // oops, shouldn't have run
            console.log(err); // evil laugh
        }
    );
```

下面的程式碼確實可以運作，然而它的運作會讓人相當意外，畢竟它本身就不是一個 Promise 物件，然而它我們使用了 `Promise.resolve(..)` 時，就像是加了一個保險，把 `thenable` 轉換成一個 Promise 物件，也因此下面的程式碼，加上 `Promise.resolve(..)` 會是更好的選擇 :

``` JavaScript
// don't just do this:
foo(42)
    .then(function(v) {
        console.log(v);
    });

// instead, do this:
Promise.resolve(foo(42))
    .then(function(v) {
        console.log(v);
    });
```

### Promise 的鏈式結構

Promise 的結構通常會寫成像下面的樣子 :

``` JavaScript
var p = Promise.resolve(21);

p
    .then(function(v) {
        console.log(v); // 21

        // fulfill the chained promise with value `42`
        return v * 2;
    })
    // here's the chained promise
    .then(function(v) {
        console.log(v); // 42
    });
```

這樣讓我們在寫 Promise 有著更明確的流程控制，也可以視需求去延遲輸出 :

``` JavaScript
function delay(time) {
    return new Promise(function(resolve, reject) {
        setTimeout(resolve, time);
    });
}

delay(100) // step 1
    .then(function STEP2() {
        console.log("step 2 (after 100ms)");
        return delay(200);
    })
    .then(function STEP3() {
        console.log("step 3 (after another 200ms)");
    })
    .then(function STEP4() {
        console.log("step 4 (next Job)");
        return delay(50);
    })
    .then(function STEP5() {
        console.log("step 5 (after another 50ms)");
    })
    ...
```

* `then(..)`

另一個值得一提的是， `resolve(..)` 所做的事情，有時 `then(..)` 方法內部會回傳 Promise，有時則是直接把值回傳出來，然而對於外部的串鏈來說都是接到 Promise，原因在於 `resovle(..)` 會持續"溶解"，並自動將內部的內容轉換成一個 Promise，所以就算回傳的東西是一個 `thenable` 物件，也會將其轉換成一個 Promise，這也是為什麼 Promise 建構式的第一個參數，要取作 `resolve(..)` 而不叫 `fulfillment` ，因為除了成功以外，還包含著其他的含意。

* `fulfillment(..)`、`reject(..)`

而在 `then(..)` 裡面會具體填入這兩個方法，如果 `fulfillment(..)` 沒有填寫，預設上它會把上一個串鏈的值繼續往下帶，而 `reject(..)` 沒有填寫，預設上會直接拋出錯誤，就像下面的語法 :

``` JavaScript
var p = new Promise(function(resolve, reject) {
    reject("Oops");
});

var p2 = p.then(
    function fulfilled() {
        // never gets here
    }
    // assumed rejection handler, if omitted or
    // any other non-function value passed
    // function(err) {
    //     throw err;
    // }
);
```
