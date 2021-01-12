## 循序的大腦

大多數人都曾經聽某個人說過 ( 甚至你自己就曾這麼說 )，"我能一心多用"。試圖表現得一心多用的效果包含幽默 ( 孩子們的拍頭揉肚子遊戲 )、平常的行為 ( 邊走邊嚼口香糖 )、和徹頭徹尾的危險 ( 開車時發微信 )。

然而實際上並不是如此，當我們模擬一心多用時，其實只是不停地在兩個行程之間切換，這就像是非同步的行為!!!

### 執行 VS 規劃

我們的大腦在規劃事情時，總是用同步的思考來進行規劃，想像再來你要做的事情 : "我得去商店，然後買些牛奶，然後去乾洗店"，注意這邊的思考完全的是以同步的方式來進行思考，中間出現的非同步請求我們可以輕鬆地解決它，然而在撰寫程式時，我們會小心翼翼地規劃所有的事情，想像一下你有辦法這樣安排事情嗎 ? 

"我得去趟商店，但是我確信在路上我會接到一個電話，於是'嗨，媽媽'，然後她開始講話，我會在GPS上搜索商店的位置，但那會花幾分鐘加載，所以我把收音機音量調小以便聽到媽媽講話，然後我發現我忘了穿夾克而且外面很冷，但沒關係，繼續開車並和媽媽說話，然後安全帶警報提醒我要係好，於是'是的，媽，我係著安全帶呢，我總是繫著安全帶！'。啊，GPS終於得到方向了，現在......"

這是種類比，在程式中我們細細地規劃每一步該怎麼做，在非同步請求時也是如此。然而對於我們來說，使用 Callback 來處理非同步請求，就像是上面的例子一樣，極其不直觀也令人困惑，原因在於我們並不是那樣思考。

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

這段程式碼常被稱作 Callback hell，也有人稱其為毀滅金字塔，但問題與縮排並沒有很大的關係。它的順序看似沒有什麼問題，可以循序的來思考，首先是 `listen` 、再來是 `setTimeout` 、接著 `ajax` 、最後是 `if` 。然而多數真實的情況是，這中間會有很多不同的雜訊，來擾亂我們對於順序上面的判別。

來看看另一個假想的情況 :

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

對於有經驗的人來說，也許他們能夠判別出其中的順序 ( A -> F -> B -> C -> E -> D )，但如果這之中，有些 `function` 不是非同步呢 ? `function` 的執行順序就會變得更為複雜。而且我們得再次強調，真實的情況遠比這些程式碼還要來得複雜。

再回到剛剛前面的 `listen` -> `setTimeout` -> `ajax` -> `if` 的例子，如果這幾個 `function` 有失敗的情況出現呢 ? 是不是又該把所有的失敗的情況給想清楚，並且寫出當失敗時應該要做什麼事情 ? 當我們事先把所有可能的情況寫出來規劃清楚，程式碼就會變得相當複雜甚至難以維護，然而這邊甚至還沒有提及 race condition 以及等待所有非同步資料回來的情況。

### 信任問題

另一個問題是 Callback 的信任問題，當我們使用類似 Ajax 的工具去後端拿資料時，通常這種工具都不是我們所開發的，他是一個第三方所提供的工具，也就是說我們將控制權轉移給第三方，這有點像是一種控制反轉 ( inversion of control )，我們的程式碼與第三方工具有個未明述的合約存在，我們期望他會按照理想完成。

然而正是因為這個第三方工具，他通常是會造成意外的一方 ! 作者在書中舉了一個極其誇張的例子，描述了 Callback 的幾個問題 :

* 太早呼叫 Callback
* 太晚呼叫 Callback ( 甚至不呼叫 )
* 呼叫太多次或是太少次
* 沒有傳入必要的參數給予 Callback
* 吞掉可能發生的錯誤與異常

這些確實是問題，面對這些問題都應該坐一些基本的防護，就像是地緣政治學的"信任但要驗證"原則。然而 Callback 並沒有提供這樣的工具，我們必須自己完成這些檢核。

#### 嘗試拯救 Callback

* 錯誤處理 :
    某些 API 提供兩個 Callback，一個用於錯誤的 Callback、一個用於成功的 Callback，就像下面的一樣 :
    ``` JavaScript
    function success(data) {
	    console.log( data );
    }

    function failure(err) {
        console.error( err );
    }

    ajax( "http://some.url.1", success, failure );
    ``` 
    然而這種做法看似幫忙處理了錯誤，不過說不定會發生兩個 Callback 都會執行的弔詭現象。

* 從未被呼叫、同步呼叫 :
    ``` JavaScript
    function timeoutify(fn, delay) {
        var intv = setTimeout(function () {
            intv = null;
            fn(new Error("Timeout!"));
        }, delay);

        return function () {
            // timeout hasn't happened yet?
            if (intv) {
                clearTimeout(intv);
                fn.apply(this, [null].concat([].slice.call(arguments)));
            }
        };
    }
    ```
    面對從未被呼叫的問題，我們嘗試做了一個 timeout 計時器，它可以幫忙取消正在發送的請求 :
    ``` JavaScript
    // using "error-first style" callback design
    function foo(err, data) {
        if (err) {
            console.error(err);
        }
        else {
            console.log(data);
        }
    }

    ajax("http://some.url.1", timeoutify(foo, 500));
    ```
    然而如果這是一個同步的請求呢 ? 把非同步混雜著同步一起使用，通常會引發令人感到崩潰的悲劇。如果我們不確定這樣的請求是布是非同步的，那這邊提供另一種工具，將同步的東西變成非同步 :

    ``` JavaScript
    function asyncify(fn) {
        var orig_fn = fn,
            intv = setTimeout(function () {
                intv = null;
                if (fn) fn();
            }, 0)
            ;

        fn = null;

        return function () {
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
    可以用下面的方式來使用 :
    ``` JavaScript
    function result(data) {
        console.log(a);
    }

    var a = 0;

    ajax("..pre-cached-url..", asyncify(result));
    a++;
    ```
    這樣雖然可以解決另一個信任問題，然而可以看看我們所新增的程式碼，這會讓我們的專案過度膨脹而且不怎麼的有效率。
    
