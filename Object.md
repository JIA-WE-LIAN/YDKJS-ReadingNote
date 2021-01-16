## 型別

* string
* number
* boolean
* null
* undefined
* object

    function 是一個子型別，也被稱作是一級函式，也就是可以被當成普通物件，但帶有可以被呼叫的行為。
    array 也是一個子型別，組織方式更有結構。

## 內建物建

* string
* number
* boolean

    上面這三個，都有包裹型態，有時候可能會寫出像 `var a = 'str'; vat len = str.length;` ，基本型別其實不帶有值，所以 JS 引擎其實有將其作轉型。而 null、undefined 則沒有包裹型別。

* Date
* object
* function
* Array
* RegExp

    上面四個通常都是使用 literal 的形式來創建，雖然使用 new 會提供較多的創建選像，然而通常會選擇簡單的形式。

* Error

    可以用new Error來創建，但通常不必要。

在 JS 中，這些都只是內建的函式，只是可以用 new 來呼叫，產生一個新的物件 :

``` JavaScript
var strPrimitive = "I am a string";
typeof strPrimitive; // "string" 
strPrimitive instanceof String; // false

var strObject = new String("I am a string");
typeof strObject; // "object" 
strObject instanceof String; // true

//考察object子類型
Object.prototype.toString.call(strObject); // [object String]
```

## 物件內容

* 取值

``` JavaScript
var myObject = {
    a: 2
};

myObject.a; // 2

myObject["a"]; // 2
```

`.` 語法通常稱作屬性存取、 `[".."]` 通常稱作鍵值存取，通常會是使用 `.` 語法，兩者差別在於， `[".."]` 語法基本可以接收任何兼容UTF-8/unicode的字串作為屬性名，而 `.` 不行。

* 屬性名

屬性名永遠是字串，不是字串的會強制轉型成字串。而 ES6 提供一種計算屬性名的寫法 :

``` JavaScript
var prefix = "foo";

var myObject = {
    [prefix + "bar"]: "hello",
    [prefix + "baz"]: "world"
};

myObject["foobar"]; // hello 
myObject["foobaz"]; // world
```

而這種最常用在 Symbol 的寫法 :

``` JavaScript
var myObject = {
    [Symbol.Something]: "hello world"
};
```

* 方法

在 JS 中，有人會說某個方法屬於哪個物件，然而嚴格來說，函式並沒有屬於哪個物件，幾乎都是參考而已，沒有擁有一個方法這種現象。

``` JavaScript
function foo() {
    console.log("foo");
}

var someFoo = foo; //對 `foo` 的變量引用

var myObject = {
    someFoo: foo
};

foo; // function foo(){..}

someFoo; // function foo(){..}

myObject.someFoo; // function foo(){..}
```

## Array

Array 採用數字索引，也就是非負整數的索引，除此之外也可以加上屬性，只是這不是一種很好的做法，畢竟 JS 引擎對 Array 有特別的優化方式。

## 複製物件

複製物件是個麻煩的問題，首先要知道自己究竟是要做淺拷貝還是深拷貝 ? 如果這個物件有著循環循環參考的情況，又該怎麼做 ? 長就以來這一直是個很大的問題，然而沒有一個統一的答案。

一種解法是像這樣，確認它是一個 JSON-safe 物件 :

``` JavaScript
var newObj = JSON.parse(JSON.stringify(someObj));
```

而另一種則是 ES6 的 `Object.assign` ，雖然它只會做第一層的拷貝，然而這也是大多數的應用情況，這寫法會把來源物件上 enumerable、自有的屬性，複製到目標物件上 ( 只是單純的 `=` 指定而已，使用definedProperty定義的內容不會複製 )。

## 屬性描述器

``` JavaScript
var myObject = {
    a: 2
};

Object.getOwnPropertyDescriptor(myObject, "a");
// {
// value: 2, 
// writable: true, 
// enumerable: true, 
// configurable: true 
// }
```

屬性除了 value 以外，還有好幾個項目可以設定。

* Writable

代表是否可以重新被覆寫 :

``` JavaScript
var myObject = {};

Object.defineProperty(myObject, "a", {
    value: 2,
    writable: false, //不可寫！
    configurable: true,
    enumerable: true
});

myObject.a = 3;

myObject.a; // 2
```

如果在嚴格模式下，JS 則會丟出 `TypeError` ，而這個 `writable: false` 跟把 `setter` 設定為空相當像，只要在手動丟出 `TypeError` 就可以了。

* Configurable

代表是否可以重新定義 `defineProperty` ( 不過 `writable` 是可以從 `true` 改成 `false` 的 )，另外連使用 `delete` 移除屬性的參考也不行 :

``` JavaScript
var myObject = {
    a: 2
};

myObject.a = 3;
myObject.a; // 3

Object.defineProperty(myObject, "a", {
    value: 4,
    writable: true,
    configurable: false, //不可配置！
    enumerable: true
});

myObject.a; // 4 
myObject.a = 5;
myObject.a; // 5

Object.defineProperty(myObject, "a", {
    value: 6,
    writable: true,
    configurable: true,
    enumerable: true
}); // TypeError
```

* Enumerable

代表是否會在列舉操作中出現 ( 像是 `for..in` 語法 )。

## 不可變

有時會希望某個屬性是不可改變的，可以將 `writable:false` 與 `configurable:false` 結合設定，然而如果此 `key` 值是參考到另一個物件，那就要在特別設定另一個物件的屬性描述器了。

``` JavaScript
myImmutableObject.foo; // [1,2,3] 
myImmutableObject.foo.push(4);
myImmutableObject.foo; // [1,2,3,4]
```

* Prevent Extensions

防止添加新的屬性，在 strict mode 會出現TypeError :

``` JavaScript
var myObject = {
    a: 2
};

Object.preventExtensions(myObject);

myObject.b = 3;
myObject.b; // undefined
```

* Seal

除了Prevent Extensions外，還把所有的屬性標記為 `configurable:false` 。

* Freeze

除了Seal外，還把所有屬性標記為 `writable:false` 。

## 取值器和設值器

* Get

當想要取得物件內部的屬性時，會用下面的方式來取得，然而這跟前面的 RHS 一樣嗎 ? 其實不太相同，它會執行內部的 get 的方法 : 

1. 看物件本身是否有這個屬性。
2. 沒有的話會往 prototype上尋找。
3. 還是沒找到的話，則會吐出一個 `undefined` 。

``` JavaScript
var myObject = {
    a: undefined
};

myObject.a; // undefined

myObject.b; // undefined
```

* Put

1. 執行 put 時，會先檢查是否有 `setter function` 。
2. 如果是 `writable:false` ，就要設定失敗，strict mode 會再拋錯
3. 不然，就依照一般情況下設定值。

如果當前的屬性還不存在，整個設定會更加複雜，這個暫且先不討論。

* 定義 getter 和 setter :

``` JavaScript
var myObject = {
    //為 `a` 定義一個getter 
    get a() {
        return 2;
    }
};

Object.defineProperty(
    myObject, //目標對象
    "b", //屬性名
    { //描述符
        //為 `b` 定義getter 
        get: function() {
            return this.a * 2
        },

        //確保 `b` 作為對象屬性出現
        enumerable: true
    }
);

myObject.a; // 2

myObject.b; // 4
```

``` JavaScript
var myObject = {
    //為 `a` 定義getter 
    get a() {
        return this._a_;
    },

    //為 `a` 定義setter 
    set a(val) {
        this._a_ = val * 2;
    }
};

myObject.a = 2;

myObject.a; // 4
```

## 存在

``` JavaScript
var myObject = {
    a: 2
};

("a" in myObject); // true 
("b" in myObject); // false

myObject.hasOwnProperty("a"); // true 
myObject.hasOwnProperty("b"); // false
```

`in` 與法會檢查整個 prototype上的物件，而 `hasOwnProperty` ，則只會檢查當前的物件，然而如果一個物件是真正的空物件 : `Object.create(null)` ，那 `hasOwnProperty` 就要這樣使用 : `Object.prototype.hasOwnProperty.call(myObject,"a")` 。

* Enumeration

當屬性存在時，此屬性並不一定可以列舉，看看下面的程式碼，除了 `for..in` 以外的幾個列舉方法 :

``` JavaScript
var myObject = {};

Object.defineProperty(
    myObject,
    "a",
    //使 `a` 可枚舉，如一般情況
    {
        enumerable: true,
        value: 2
    }
);

Object.defineProperty(
    myObject,
    "b",
    //使 `b` 不可枚舉
    {
        enumerable: false,
        value: 3
    }
);

myObject.propertyIsEnumerable("a"); // true 
myObject.propertyIsEnumerable("b"); // false

Object.keys(myObject); // ["a"] 
Object.getOwnPropertyNames(myObject); // ["a", "b"]
```

* Iteration

`for..in` 可以迭代所有屬性，如果要迭代值呢 ? 如果是 Array 確實有幾個方法可以迭代，像是 `forEach(..)` 、 `every(..) return false會結束` 、 `some(..) return true會結束` 。

相比起 Array，不要直接依賴物件迭代的順序，物件的順序是不確定的，而且端看實作的引擎為何。

如果想直接迭代值，可以使用 `for..of` 語法 :

``` JavaScript
var myArray = [1, 2, 3];

for (var v of myArray) {
    console.log(v);
}
// 1
// 2 
// 3
```

如果想進行手動迭代，可以使用 `Symbol.iterator` 取得 `@@iterator` 的內部屬性，而 `@@iterator` 則是回傳迭代器的方法。

``` JavaScript
var myArray = [1, 2, 3];
var it = myArray[Symbol.iterator]();

it.next(); // { value:1, done:false } 
it.next(); // { value:2, done:false } 
it.next(); // { value:3, done:false } 
it.next(); // { done:true }
```

雖然一般的物件沒有迭代器，然而還是可以自行定義 :

``` JavaScript
var myObject = {
    a: 2,
    b: 3
};

Object.defineProperty(myObject, Symbol.iterator, {
    enumerable: false,
    writable: false,
    configurable: true,
    value: function() {
        var o = this;
        var idx = 0;
        var ks = Object.keys(o);
        return {
            next: function() {
                return {
                    value: o[ks[idx++]],
                    done: (idx > ks.length)
                };
            }
        };
    }
});

//手動迭代 `myObject`
var it = myObject[Symbol.iterator]();
it.next(); // { value:2, done:false } 
it.next(); // { value:3, done :false } 
it.next(); // { value:undefined, done:true }

//用 `for..of` 迭代 `myObject`
for (var v of myObject) {
    console.log(v);
}
// 2
// 3
```

通常會用 `Object.defineProperty(..)` 來定義，畢竟會設定 `enumerable: false` 。而這種自訂義的做法相當好用，舉個例子，一個 `Pixel（像素）` 對象列表（擁有 `x` 和 `y` 的坐標值）可以根據距離原點 `(0,0)` 的直線距離決定它的迭代順序，或者過濾掉那些"太遠"的點，等等。只要你的迭代器從 `next()` 調用返回期望的 `{ value: .. }` 返回值，並在迭代結束後返回一個 `{ done: true }` 值，ES6 的 `for..of` 循環就可以迭代它。
