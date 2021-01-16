## 原型的誤解

如果想嘗試在 JS 中做出跟類別導向相似的程式碼，會有好幾種作法，然而所帶來的好處，永遠都會比壞處來的少 :

* Explicit Mixins :

``` JavaScript
//大幅簡化的 `mixin(..)` 示例：
function mixin(sourceObj, targetObj) {
    for (var key in sourceObj) {
        //僅拷貝非既存內容
        if (!(key in targetObj)) {
            targetObj[key] = sourceObj[key];
        }
    }

    return targetObj;
}

var Vehicle = {
    engines: 1,

    ignition: function() {
        console.log("Turning on my engine.");
    },

    drive: function() {
        this.ignition();
        console.log("Steering and moving forward!");
    }
};

var Car = mixin(Vehicle, {
    wheels: 4,

    drive: function() {
        Vehicle.drive.call(this);
        console.log("Rolling on all " + this.wheels + " wheels!");
    }
});
```

為了展現多型，在 `Vehicle` 和 `Car` 新增了一模一樣的特性 `drive` ，而在 `Car` 中為了呼叫 `Vehicle` 的 `drive` 函式，使用了 `Vehicle.drive.call(this);` ，這種手動呼叫 `super` 的方法，會增加程式碼的維護成本與閱讀性。

有人相對應的改成另一種作法 :`

``` JavaScript
//另一種mixin，對覆蓋不太"安全"
function mixin(sourceObj, targetObj) {
    for (var key in sourceObj) {
        targetObj[key] = sourceObj[key];
    }

    return targetObj;
}

var Vehicle = {
    // ... 
};

//首先，創建一個空對象
//將Vehicle的內容拷貝進去
var Car = mixin(Vehicle, {});

//現在拷貝Car的具體內容
mixin({
    wheels: 4,

    drive: function() {
        // ... 
    }
}, Car);
```

這幾行程式碼似乎幫忙做了拷貝，然而實際上並沒有，它只是添加了 JS 的函式參考而已，**類別版本的物件導向，即使是父子繼承關係，只要用了 `new` 關鍵字，兩者的方法就會是獨立的!!! 但 JS 並不完全獨立，而是單純的參考同一函式。**
``` JavaScript     
for (var key in sourceObj) {

    targetObj[key] = sourceObj[key];

}

``` 
除了前面的兩個之外，書中還展現了另外兩個寫法 ( Parasitic Inheritance、Implicit Mixins )，雖說都是想嘗試模擬出類別的概念，然而 JS 根本的原理並不建議使用類別的方式去思考。

## 原型

1. 
當想調用一個物件的屬性時，會呼叫預設的 `get` 方法，它所做的事情，會是找尋當前的物件看是否有這個屬性，沒有的話會繼續往它的 prototype 上尋找 ( 不過如果涉及 ES6 的 Proxy，這邊的東西就不太適用了 )，因此會有遮蔽的效果出現，如果下面的物件已經找到，就不會繼續往上尋找 :
``` JavaScript
var anotherObject = {
    a: 2
};

//創建一個鏈接到 `anotherObject` 的對象
var myObject = Object.create(anotherObject);

myObject.a;  // 2
```

2. 
當使用了 `for..in` 的語法時，任何一個被 prototype chain 的元素都會被迭代出來 ( 只要是 enumerable )，而如果是用 `in` 語法，則不論是不是 enumerable 都會被檢查出來。

``` JavaScript
var anotherObject = {
    a: 2
};

//創建一個鏈接到 `anotherObject` 的對象
var myObject = Object.create(anotherObject);

for (var k in myObject) {
    console.log("found: " + k);
}
//找到: a

("a" in myObject); // true
```

3. 
Object.prototype 是整個 prototype 的終點，各種常使用的工具也都會在這邊看到，像是 `toString()` 、 `valueOf()` 、 `hasOwnProperty()` 、 `isPrototypeOf()` 。

4. 
剛剛前面的第一點提到會有遮蔽效果，然而其實這只是其中一點而已，如果真的要細看，會分成下面三種情況 :

* 如果一個普通的名為foo的數據訪問屬性在 `[[Prototype]]` 鏈的高層某處被找到，而且沒有被標記為 `writable:false` ，那麼一個名為 `foo` 的新屬性就直接添加到 `myObject` 上，形成一個遮蔽屬性。

* 如果一個 `foo` 在 `[[Prototype]]` 鏈的高層某處被找到，但是它被標記為 `writable:false` ，那麼設置既存屬性和在 `myObject` 上創建遮蔽屬性都是不允許的。如果代碼運行在 `strict mode` 下，一個錯誤會被拋出。否則，這個設置屬性值的操作會被無聲地忽略。不論怎樣，沒有發生遮蔽。

* 如果一個 `foo` 在 `[[Prototype]]` 鏈的高層某處被找到，而且它是一個 `setter` ，那麼這個 `setter` 總是被調用。沒有 `foo` 會被添加到（也就是遮蔽在） `myObject` 上，這個 `foo`  `setter` 也不會被重定義。

通常都會發生大家的認知都是發生第一項，然而其實還有第二、三項可能會發生，如果問題出現在第二、三項時，就得利用 `Object.definedProperty(..)` 來定義屬性，不能單純的使用 `=` ( 第二種情況相當令人詫異，不過如果用類別的方式去思考這個繼承問題，可能就沒那麼古怪了，然而要知道 JS 的運作模式實際上跟類別不一樣，而且更加詭異的是，居然可以使用 `defineProperty` 來設定值而不是完全禁止 )。

另外操作可能會發生遮蔽的效果的屬性，必須相當小心 :

``` JavaScript
var anotherObject = {
    a: 2
};

var myObject = Object.create(anotherObject);

anotherObject.a; // 2 
myObject.a; // 2

anotherObject.hasOwnProperty("a"); // true 
myObject.hasOwnProperty("a"); // false

myObject.a++; //噢，隱式遮蔽！

anotherObject.a; // 2 
myObject.a; // 3

myObject.hasOwnProperty("a"); // true
```

## 很像類別的函式

首先要先知道，所有的 JS 函式都有一個不可列舉的屬性 prototype，這個 prototype 可以指向任意的屬性 :

``` JavaScript
function Foo() {
    // ... 
}

Foo.prototype; // { }
```

那在預設的情況下，它會做些什麼 ? 來看看 `new` 關鍵字 :

``` JavaScript
function Foo() {
    // ... 
}

var a = new Foo();

Object.getPrototypeOf(a) === Foo.prototype; // true
```

以類別的思考模式來看，使用 `new` 關鍵字就像是按照一個模板，把一個物件給刻劃出來，然而在 JS 中並沒有這樣的模板，存在的是一個特別的物件 prototype，並且使用這個物件相互連接，這是 `new` 關鍵字的一個特別副作用。

那一般的情況之下，可以這樣嗎 ? 可以!!! 使用 `Object.create()` 可以辦到這件事情。

``` mermaid
graph TD
	Foo.prototype---a1
    Foo.prototype---a2
    Foo.prototype---Bar.prototype
    Bar.prototype---b1
    Bar.prototype---b2
```

## 建構子

JS 有預設一種屬性在 prototype 上，讓人覺得 function 似乎是個建構子 :

``` JavaScript
function Foo() {
    // ... 
}

Foo.prototype.constructor === Foo; // true

var a = new Foo();
a.constructor === Foo; // true
```

問題一直都在於 new 關鍵字，這會讓人覺得這裡似乎有著建構子，然而只是不同的呼叫模式而已，另外 `a.constructor === Foo` 更是一行會讓人困惑的程式碼，其實 `a` 本身沒有 `constructor` 屬性，這屬性其實是在 `Foo.prototype` 上找到的，但 `Foo.prototype` 是在 `a` 的 prototype chain 上，所以就誤打誤撞變成了 `true` 。

如果把程式碼改成這樣，就會很明顯地看到這個誤打誤撞的結果 :

``` JavaScript
function Foo() {
    /* .. */
}

Foo.prototype = {
    /* .. */
}; //創建一個新的prototype對象

var a1 = new Foo();
a1.constructor === Foo; // false! 
a1.constructor === Object; // true!
```

當然這樣做，還是有辦法自行把 constructor 修補回來 :

``` JavaScript
function Foo() {
    /* .. */
}

Foo.prototype = {
    /* .. */
}; //創建一個新的prototype對象

//需要正確地"修復"丟失的 `.constructor`
//新對像上的屬性以 `Foo.prototype` 的形式提供。
Object.defineProperty(Foo.prototype, "constructor", {
    enumerable: false,
    writable: true,
    configurable: true,
    value: Foo //使 `.constructor` 指向 `Foo`
});
```

然而可以看到這些屬性，其實都是可以任意去修改的，簡而言之是個算是相當不靠譜的屬性。

## 原型式繼承

講到繼承時，多數人所想到的是類別式繼承，然而 JS 所呈現的，完全不是這麼一回事，所有的東西都是物件，來看看一個典型的 JS 原型式繼承 :

``` JavaScript
function Foo(name) {
    this.name = name;
}

Foo.prototype.myName = function() {
    return this.name;
};

function Bar(name, label) {
    Foo.call(this, name);
    this.label = label;
}

//這裡，我們創建一個新的 `Bar.prototype` 鏈接鏈到 `Foo.prototype`
Bar.prototype = Object.create(Foo.prototype);

//注意！現在 `Bar.prototype.constructor` 不存在了，
//如果你有依賴這個屬性的習慣的話，它可以被手動“修復”。

Bar.prototype.myLabel = function() {
    return this.label;
};

var a = new Bar("a", "obj a");

a.myName(); // "a" 
a.myLabel(); // "obj a"
```

可以看到，為了建立兩個物件之間的連結，使用了 `Object.create(..)` 的語法，或許有人會把程式碼改成這樣 :

``` JavaScript
//不會如你期望的那樣工作! 
Bar.prototype = Foo.prototype;

//會如你期望的那樣工作
//但會帶有你可能不想要的副作用:( 
Bar.prototype = new Foo();
```

首先第一個完全是錯誤的寫法，它把兩個函式的參照寫在一起，第二個寫法雖然會達到效果，然而寫下 `new` 關鍵字時，就會執行 `function` 的內容，造成一些副作用 ( 像是寫 log、 `this` 關鍵字...... )。 

而 ES6 提供了一個新語法也可以用來修改 prototype，或者也可以使用 `__proto__` 來獲取上方的 prototype :

``` JavaScript
// ES6以前
//扔掉默認既存的 `Bar.prototype`
Bar.prototype = Object.create(Foo.prototype);

// ES6+ 
//修改既存的 `Bar.prototype`
Object.setPrototypeOf(Bar.prototype, Foo.prototype);
```

## instanceof

``` JavaScript
function Foo() {
    // ... 
}

Foo.prototype.blah = ...;

var a = new Foo();

a instanceof Foo; //true
```

在這段程式碼裡面，instanceof 所做的事情，是回答在 `a` 的整個原型串鏈中，Foo.prototype 是否有出現過，然而這個語法比較像是存在於類別導向程式語言的語法，語意其實有些模糊。

有個特別的點是，如果使用 `bind(..)` 語法得到的 function，其實是沒有 prototype 這個屬性的，因此使用 new 呼叫時，會回到原本的 function。

更進階的可能會採用這幾種語法 :

``` JavaScript
b.isPrototypeOf(c);
Object.getPrototypeOf(a) === Foo.prototype; // true
a.__proto__ === Foo.prototype; // true( dunder proto不是標準之一 )
```

而 dunder proto 其實就是底下的 polyfill :

``` JavaScript
Object.defineProperty(Object.prototype, "__proto__", {
    get: function() {
        return Object.getPrototypeOf(this);
    },
    set: function(o) {
        // ES6的setPrototypeOf(..) 
        Object.setPrototypeOf(this, o);
        return o;
    }
});
```

## Object.create(..)

而 `Object.create(..)` 一直都是可以讓 prototype 得以實踐的英雄，它將兩個物件給鏈結再一起。

`Object.create(null)` 那跟 `null` 進行連結，究竟有何好處，好處在於這樣的物件完全不跟任何人有勾搭，有些時候可以拿它來儲存資料，就像是個扁平化的資料庫。

那如果想 `polyfill` 這個功能呢，有個部分 polyfill 的寫法 :

``` JavaScript
if (!Object.create) {
    Object.create = function(o) {
        function F() {}
        F.prototype = o;
        return new F();
    };
}
```

## 把prototype作為候補 ?

也許在看到委派後，會想嘗試看看這樣的程式碼 :

``` JavaScript
var anotherObject = {
    cool: function() {
        console.log("cool!");
    }
};

var myObject = Object.create(anotherObject);

myObject.cool(); // "cool!"
```

然而這件事情有些特別，畢竟 `myObject` 並沒有 `cool` 這個屬性，在維護上可能會讓人有些訝異，這種寫法也較為少見，因此建議改以下面的寫法 :

``` JavaScript
var anotherObject = {
    cool: function() {
        console.log("cool!");
    }
};

var myObject = Object.create(anotherObject);

myObject.doCool = function() {
    this.cool(); // internal delegation! 
};

myObject.doCool(); // "cool!"
```
