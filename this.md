## this 的混淆之處

`this` 代表的意思，會容易聯想到自身的意思，然而不是這麼回事 :

``` JavaScript
function foo(num) {
    console.log("foo: " + num);

    //追蹤 `foo` 被調用了多少次
    this.count++;
}

foo.count = 0;

var i;

for (i = 0; i < 10; i++) {
    if (i > 5) {
        foo(i);
    }
}
// foo: 6 
// foo: 7 
// foo: 8 
// foo: 9

// `foo` 被調用了多少次?
console.log(foo.count); // 0 ……?
```

碰到這樣的問題後，有人會將想改變的參數傳進來，避免使用 `this` ，然而實際上只是逃避 `this` 的問題，在更進一步的，如果想寫出類似遞迴的程式碼，也許會用 `this` 來呼叫 `function` 自己，這也會拿到失敗的結果 ( 要讓 `function` 呼叫自己，最好是給 `function` 一個適當的名稱，不然得使用棄用的 `arguments.callee` 了 )，最後看看下面的程式碼 :

``` JavaScript
function foo() {
    var a = 2;
    this.bar();
}

function bar() {
    console.log(this.a);
}

foo(); //undefined
```

這段程式碼嘗試著把 `this` 和 lexical scope 兩者建立一座橋梁，不過這種東西在 JS 並不存在。

## this 的四種規則

* default binding

``` JavaScript
function foo() {
    console.log(this.a);
}

var a = 2;

foo(); // 2
```

第一個規則，就是不是套用其他規則時所預設的， `this` 會指向全域物件 `window` ，如果是在 `strict mode` 會是 `undefined` ，不過如果 `strict mode` 和一般模式混用時 ( 這種情況比較少見，通常會是引入第三方套件才會有這樣的情況 )，只要 `function` 內容不在 `strict mode` 下，就還會是 `window` :

``` JavaScript
function foo() {
    console.log(this.a);
}

var a = 2;

(function() {
    "use strict";

    foo(); // 2 
})();
```

* implict binding

``` JavaScript
function foo() {
    console.log(this.a);
}

var obj = {
    a: 2,
    foo: foo
};

obj.foo(); // 2
```

在這種情境下，可以先注意到 `obj` 中的 `foo` 只是對函式的參考而已，它並不是真的擁有這個函式，而這樣的綁定會自然而然地綁到 `function foo` 上面。

另外還有兩個可以注意到的點，第一個是它只會抓取最上層的物件 ( 這樣想想合理，畢竟只是函式的參考 ) :

``` JavaScript
function foo() {
    console.log(this.a);
}

var obj2 = {
    a: 42,
    foo: foo
};

var obj1 = {
    a: 2,
    obj2: obj2
};

obj1.obj2.foo(); // 42
```

第二個是，當把這種 `obj.foo` 當成參數或是 `callback` 傳遞時，其實就會失去 `implict binding` 的效果，退回到原本 `default binding` :

``` JavaScript
function foo() {
    console.log(this.a);
}

var obj = {
    a: 2,
    foo: foo
};

var bar = obj.foo; //函數引用！

var a = "oops, global"; // `a` 也是一個全局對象的屬性

bar(); // "oops, global"
```

如果只是對於函式的參考，其實是沒有效用的， `this` 的判斷只能端看執行時的情況。

* explict binding

明確綁定，也就是 `call` 、 `bind` 、 `apply` ，這三個方法隸屬於 `Function.prototype` 上，因此所有的函式都可以呼叫 :

``` JavaScript
function foo() {
    console.log(this.a);
}

var obj = {
    a: 2
};

foo.call(obj);
```

`call` 和 `apply` 都是直接執行函式的結果，第一個參數都是放入想指定為 `this` 的物件，而 `call` 後面的參數是放入執行那個 `function` 所需要的參數， `apply` 則是將這些參數陣列化，因此它總共只會有兩個參數而已。

`bind` 則是回傳一個 `function` ，只是它直接將 `this` 綁死在第一個參數上。

另外有些函式 API 甚至直接提供而外的參數，讓使用者可以直接指定 `this` 物件，像是 forEach 的第二個參數。

* new binding

最後一項是 new binding，它總共會做下面四件事情 :

1. 會有一個無中生有的物件被創建出來
2. 這個新建構的物件帶有 `[[Prototype]]` 連結
3. 這個新建構的物件，會被當作 this 的物件
4. 除非函式自己提供物件，不然以 new 呼叫的程式，會自動回傳這個物件

``` JavaScript
function foo(a) {
    this.a = a;
}

var bar = new foo(2);
console.log(bar.a); // 2
```

## this 的優先順序

下一個就是探索這幾種 this 混用在一起，又該如何判斷，我們可以把 default binding 放在最後，畢竟它是預設值 :

* implict binding vs. explict binding(優先級高)

``` JavaScript
function foo() {
    console.log(this.a);
}

var obj1 = {
    a: 2,
    foo: foo
};

var obj2 = {
    a: 3,
    foo: foo
};

obj1.foo(); // 2 
obj2.foo(); // 3

obj1.foo.call(obj2); // 3 
obj2.foo.call(obj1); // 2
```

* implict binding vs. new binding(優先級高)

``` JavaScript
function foo(something) {
    this.a = something;
}

var obj1 = {
    foo: foo
};

var obj2 = {};

obj1.foo(2);
console.log(obj1.a); // 2

obj1.foo.call(obj2, 3);
console.log(obj2.a); // 3

var bar = new obj1.foo(4);
console.log(obj1.a); // 2 
console.log(bar.a); // 4
```

* explict binding vs. new binding(優先級高)

因為 `call` 、 `apply` 不能和 `this` 並用，因此改用 `bind` 來進行測試 :

``` JavaScript
function foo(something) {
    this.a = something;
}

var obj1 = {};

var bar = foo.bind(obj1);
bar(2);
console.log(obj1.a); // 2

var baz = new bar(3);
console.log(obj1.a); // 2 
console.log(baz.a); // 3
```

可以看到 new binding，還是會獨立創造出物件 ( 如果 bind 綁定優先級比 new 還要高， `baz` 和 `obj1` 就會是一模一樣的物件了!!! )。

這種特性有何用處，這實際上是 partial application ( 是 currying 的一種子集合，在作者的另一本書中 functional light JS 有詳細的提到 )，他可以決定部分的參數，然而卻沒有強制綁定內部的 `this` :

``` JavaScript
function foo(p1, p2) {
    this.val = p1 + p2;
}

//在這裡使用 `null` 是因為在這種場景下我們不關心 `this` 的硬綁定
//而且反正它將會被 `new` 調用覆蓋掉！
var bar = foo.bind(null, "p1");

var baz = new bar("p2");

baz.val; // p1p2
```

比較後的優先級如下 :

1. new binding
2. explict binding
3. implict binding
4. default binding

## 例外

* 忽略this

``` JavaScript
function foo() {
    console.log(this.a);
}

var a = 2;

foo.call(null); // 2
```

這種在第一個參數放入 `null` 的寫法，主要是想忽略 `call` 、 `apply` 、 `bind` 的第一個佔位，然而這有什麼用處，在 ES6 之前，使用 `apply` 方法可以讓陣列當作參數，而 `bind` 則可以實踐柯理化 :

``` JavaScript
function foo(a, b) {
    console.log("a:" + a + ", b:" + b);
}

//將數組散開作為參數
foo.apply(null, [2, 3]); // a:2, b:3

//用 `bind(..)` 進行柯里化
var bar = foo.bind(null, 2);
bar(3); // a:2, b:3
```

然而如果這個函式有使用到 `this` ，這就會不經意的變成 `window` 造成問題 ( 退化成 default binding )，因此作者提供一種作法，傳入一個真正的空物件 :

``` JavaScript
function foo(a, b) {
    console.log("a:" + a + ", b:" + b);
}

//我們的DMZ(demilitaried zone，非軍事區)空對象
var ø = Object.create(null);

//將數組散開作為參數
foo.apply(ø, [2, 3]); // a:2, b:3

//用 `bind(..)` 進行currying 
var bar = foo.bind(ø, 2);
bar(3); // a:2, b:3
```

* 間接傳遞

這邊寫了一個傳遞的語法，然而要知道這邊只是傳遞 `function` 的引用，不會把 `this` 也進行傳遞 :

``` JavaScript
function foo() {
    console.log(this.a);
}

var a = 2;
var o = {
    a: 3,
    foo: foo
};
var p = {
    a: 4
};

o.foo(); // 3 
(p.foo = o.foo)(); // 2
```

* 軟化綁定

相對起 `bind` 的硬綁定，雖然它防止函式不經意的退到全域物件，然而這大大降低了函式的靈活性，因此就出現了 Softening Binding 的一種寫法 :

``` JavaScript
if (!Function.prototype.softBind) {
    Function.prototype.softBind = function(obj) {
        var fn = this,
            curried = [].slice.call(arguments, 1), // arguments是array like物件，不得直接使用slice
            bound = function bound() {
                return fn.apply(
                    (!this ||
                        (typeof window !== "undefined" &&
                            this === window) ||
                        (typeof global !== "undefined" &&
                            this === global)
                    ) ? obj : this,
                    curried.concat.apply(curried, arguments) // 這個arguments 和上面的arguments是不同的 !!! 屬於不同scope
                );
            };

        bound.prototype = Object.create(fn.prototype);
        return bound;
    };
}
```

``` JavaScript
function foo() {
    console.log("name: " + this.name);
}

var obj = {
        name: "obj"
    },
    obj2 = {
        name: "obj2"
    },
    obj3 = {
        name: "obj3"
    };

var fooOBJ = foo.softBind(obj);

fooOBJ(); // name: obj

obj2.foo = foo.softBind(obj);
obj2.foo(); // name: obj2 <----看!!!

fooOBJ.call(obj3); // name: obj3 <----看!

setTimeout(obj2.foo, 10); // name: obj <----退回到軟綁定
```

* lexical scope 的 this

ES6 引入了一種不適用於這些規則特殊的函數箭頭函式，它的 this 指向閉包它的函式的 this，直接完全參照 lexical scope 的規則 :

``` JavaScript
function foo() {
    //返回一個箭頭函數
    return (a) => {
        //這裡的 `this` 繼承自 `foo()`
        console.log(this.a);
    };
}

var obj1 = {
    a: 2
};

var obj2 = {
    a: 3
};

var bar = foo.call(obj1);
bar.call(obj2); // 2,不是3!
```

在 ES6 之前，已經有種廣為人知的寫法，跟箭頭函式的精神沒有區別 :

``` JavaScript
function foo() {
    var self = this; //捕獲上層 `this`
    setTimeout(function() {
        console.log(self.a);
    }, 100);
}

var obj = {
    a: 2
};

foo.call(obj); // 2
```

一個函式內，確實可以包含這兩種風格的程式碼 ( 依照一般的 `this` 規則或是按照語彙範疇 )，但這種混淆的寫法，會讓程式碼變得難以維護，建議就只挑選其中一種即可。
