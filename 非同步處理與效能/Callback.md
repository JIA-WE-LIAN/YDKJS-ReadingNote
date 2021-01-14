## 循序的大腦

或許有人覺得自己可以一心多用，然而實際上並不可能，嘗試模擬一心多用時，其實是不停地在兩個行為之間切換，就像是非同步的行為，這是個重要的類比與影響，會讓我們覺得 Callback 並不好用的其中一大原因也在這裡。

### 執行 VS 規劃

大腦在規劃事情時，總是用同步的思考來進行，想像再來要做的事情 : "去一趟商店，買一些牛奶，再去乾洗店......"，注意這邊的思考是以同步的方式在規劃，實際進行時中間應該會出現各種非同步請求，我們可以輕鬆地解決它，然而撰寫程式時，必須小心翼翼地規劃所有的情況與可能，想像一下我們有辦法這樣安排事情嗎 ? 

"我得去趟商店，但是我確信在路上我會接到一個電話，於是'嗨，媽媽'，然後她開始講話，我會在 GPS 上搜索商店的位置，但那會花幾分鐘加載，所以我把收音機音量調小以便聽到媽媽講話，然後我發現我忘了穿夾克而且外面很冷，但沒關係，繼續開車並和媽媽說話，然後安全帶警報提醒我要繫好，於是'是的，媽，我繫著安全帶呢'。啊，GPS終於得到方向了，現在......"

在程式中我們細細地規劃每一步該怎麼做，在非同步請求時也是如此。然而對於我們來說，使用 Callback 來處理非同步請求，就像是上面的例子一樣，極其不直觀也令人困惑，原因在於我們並不是那樣思考。

### 巢狀或鏈串的 Callback

``` JavaScript
listen("click", function handler(evt) {
    setTimeout(function request() {
        ajax("http://some.url.1", function response(text) {
            if (text == "hello") {
                handler();
            } else if (text == "world") {
                request();
            }
        });
    }, 500);
});
```

這段程式碼常被稱作 Callback hell，有人稱其為毀滅金字塔，但問題與縮排並不大。它的順序看似沒有什麼問題，可以循序的來思考，首先是 `listen` 、再來是 `setTimeout` 、接著 `ajax` 、最後是 `if` 。然而多數真實的情況是，這中間會有很多不同的雜訊，來擾亂我們對於順序上面的判別。

看看一個假想的情況 :

``` JavaScript
doA(function() {
    doB();

    doC(function() {
        doD();
    })

    doE();
});

doF();
```

對於有經驗的人來說，也許能夠判別出其中的順序 ( A -> F -> B -> C -> E -> D )，但如果這之中有些 `function` 是同步呢 ? `function` 的執行順序就會變得更為複雜。而且得再次強調，真實的情況遠比這些程式碼還要來得複雜。

再回到剛剛前面的 `listen` -> `setTimeout` -> `ajax` -> `if` 的例子，如果有失敗的情況出現呢 ? 是不是又該把所有的失敗的情況給想清楚，並且寫出當失敗時應該要做什麼事情 ? 當事先把所有可能的情況寫出來規劃清楚，程式碼就會變得相當複雜甚至難以維護，然而這邊甚至還沒提及 gate 或 latch 這兩種非同步問題。

### 信任問題

另一個問題是 Callback 的信任問題，當使用類似 Ajax 的工具去後端拿資料時，通常這種工具不是由我們所開發，它是第三方所提供的工具，也就是說我們將控制權轉移給第三方，這有點像是控制反轉 ( inversion of control )，我們的程式碼與第三方工具有個未明述的合約存在，我們期望他會按照理想完成。

然而因為第三方工具，它通常是會造成意外的地方，作者在書中舉了個誇張的例子，描述 Callback 的幾個問題 :

* 太早呼叫 Callback
* 太晚呼叫 Callback ( 甚至不呼叫 )
* 呼叫太多次或是太少次
* 沒有傳入必要的參數給予 Callback
* 吞掉可能發生的錯誤與異常

這些確實是問題，面對這些問題都該做些基本防護。然而 Callback 並沒有提供這樣的工具，必須自行完成這些檢核。

#### 嘗試拯救 Callback

* 錯誤處理 :
    某些 API 提供兩個 Callback，一個用於錯誤的 Callback、一個用於成功的 Callback，就像下面的一樣 :

    ``` JavaScript
    function success(data) {
        console.log(data);
    }

    function failure(err) {
        console.error(err);
    }

    ajax("http://some.url.1", success, failure);
    ```

    然而這種做法看似幫忙處理了錯誤，不過說不定會發生兩個 Callback 都會執行的弔詭現象 ( 將控制權轉移給第三方，誰也不知道它會怎麼做 )。

* 從未被呼叫、同步呼叫 :
    面對從未被呼叫的問題，我們嘗試做了一個 timeout 計時器，它可以幫忙取消正在發送的請求 :

    ```JavaScript
    function timeoutify(fn, delay) {
        var intv = setTimeout(function() {
            intv = null;
            fn(new Error("Timeout!"));
        }, delay);

        return function() {
            // timeout hasn't happened yet?
            if (intv) {
                clearTimeout(intv);
                fn.apply(this, [null].concat([].slice.call(arguments)));
            }
        };
    }
    ```

    ``` JavaScript
    // using "error-first style" callback design
    function foo(err, data) {
        if (err) {
            console.error(err);
        } else {
            console.log(data);
        }
    }

    ajax("http://some.url.1", timeoutify(foo, 500));
    ```

    然而如果是同步的請求呢 ? 非同步混雜著同步一起使用，通常會造成很大的問題。如果不確定這樣的請求是否為非同步，那這邊提供另一種工具，可以將同步的 function 變成非同步 :

    ``` JavaScript
    function asyncify(fn) {
        var orig_fn = fn,
            intv = setTimeout(function() {
                intv = null;
                if (fn) fn();
            }, 0);

        fn = null;

        return function() {
            // firing too quickly, before `intv` timer has fired to
            // indicate async turn has passed?
            if (intv) {
                fn = orig_fn.bind.apply(
                    orig_fn,
                    // add the wrapper's `this` to the `bind(..)`
                    // call parameters, as well as currying any
                    // passed in parameters
                    [this].concat([].slice.call(arguments))
                );
            }
            // already async
            else {
                // invoke original function
                orig_fn.apply(this, arguments);
            }
        };
    }
    ```
    ``` JavaScript
    function result(data) {
        console.log(a);
    }

    var a = 0;

    ajax("..pre-cached-url..", asyncify(result));
    a++;
    ```

    雖然解決了另一個信任問題，然而可以看看新增的程式碼，這會讓專案過度膨脹而且變得不怎麼有效率。
