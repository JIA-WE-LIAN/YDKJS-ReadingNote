## 行為委派

在類別理論中，通常會做一件事情，就是將通用或是基礎的部分，抽取到父類別，然後在子類別去加以覆寫 :

``` Java
class Task {
	id;

	// `Task()` 构造器
	Task(ID) { id = ID; }
	outputTask() { output( id ); }
}

class XYZ inherits Task {
	label;

	// `XYZ()` 构造器
	XYZ(ID,Label) { super( ID ); label = Label; }
	outputTask() { super(); output( label ); }
}

class ABC inherits Task {
	// ...
}
```

當定義好類別之後，可以去實體化這些類別，並直接操作這些類別的方法。

## 委派理論

首先會定義一個 `Task` 物件，然後定義其他物件 ( 假設是 `XYZ` )以及它該專屬處理的任務，我們讓這些物件連結到 `Task` 物件，達成委派的效果。基本上可以想像成這兩個物件是兄弟或是同儕物件，不須經過類別的複製動作，這兩個物件是分離的 :

``` JavaScript
var Task = {
    setID: function(ID) {
        this.id = ID;
    },
    outputID: function() {
        console.log(this.id);
    }
};

//使 `XYZ` 委託到 `Task`
var XYZ = Object.create(Task);

XYZ.prepareTask = function(ID, Label) {
    this.setID(ID);
    this.label = Label;
};

XYZ.outputTaskDetails = function() {
    this.outputID();
    console.log(this.label);
};

// ABC = Object.create( Task ); 
// ABC ... = ...
```

這是一種極端前大的模式，重點有三項 :

1. 一般來說，資料的特性會寫在受委派者身上，以上面的例子就是 `XYZ` 。
2. 在類別模式裡面，會鼓勵相同的方法名稱，然而在委派模式中，會盡量避免這件事情，不然會發生遮蔽效果。而且實質上，每個物件都有自己**專屬**的行為，也就是說不太可能會有重複的命名。
3. 在上面的例子中，雖然 `XYZ` 的方法帶有 `this` 關鍵字，然而真的呼叫時，其實會綁定到 `Task` 上 ( JS是對函式的參考 )。

* JS 並不允許兩個物件進行相互委派這件事情。

* 在 chrome 瀏覽器中，它會替建構出來的物件給予名字，就像下面的例子，有人認為這是利用 `constructor.name` 的屬性找到的 :

``` JavaScript
function Foo() {}

var a1 = new Foo();

a1.constructor; // Foo(){} 
a1.constructor.name; // "Foo"
```

然而，當我們把它這個屬性改掉，卻不是如此 :

``` JavaScript
function Foo() {}

var a1 = new Foo();

Foo.prototype.constructor = function Gotcha() {};

a1.constructor; // Gotcha(){} 
a1.constructor.name; // "Gotcha"

a1; // Foo {}
```

但如果這樣寫，卻可以直接改變內部的內容 :

``` JavaScript
var Foo = {};

var a1 = Object.create(Foo);

a1; // Object {}

Object.defineProperty(Foo, "constructor", {
    enumerable: false,
    value: function Gotcha() {}
});

a1; // Gotcha {}
```

這些細節對於使用類別導向的想法來說，其實算是重要的一環，然而使用委派的想法，就變得一點也不重要，畢竟誰在乎建構子是誰。

* 作者認為 OLOO 風格更加完美支援關注點分離，讓初始化與建構這兩件事情完全分離，然而個人認為並不一定。

## 心智圖

委派可以讓整個心智圖的理解變得更加簡單，第一個是類別理論的心智圖 :

``` JavaScript
function Foo(who) {
    this.me = who;
}
Foo.prototype.identify = function() {
    return "I am " + this.me;
};

function Bar(who) {
    Foo.call(this, who);
}
Bar.prototype = Object.create(Foo.prototype);

Bar.prototype.speak = function() {
    alert("Hello, " + this.identify() + ".");
};

var b1 = new Bar("b1");
var b2 = new Bar("b2");

b1.speak();
b2.speak();
```

![image](/images/BD-1.png)

![image](/images/BD-2.png)

下一個是委派理論的心智圖 :

``` JavaScript
var Foo = {
    init: function(who) {
        this.me = who;
    },
    identify: function() {
        return "I am " + this.me;
    }
};

var Bar = Object.create(Foo);

Bar.speak = function() {
    alert("Hello, " + this.identify() + ".");
};

var b1 = Object.create(Bar);
b1.init("b1");
var b2 = Object.create(Bar);
b2.init("b2");

b1.speak();
b2.speak();
```

![image](/images/BD-3.png)

## 展示兩種的差別

類別化 :

``` JavaScript
//父類
function Controller() {
    this.errors = [];
}
Controller.prototype.showDialog = function(title, msg) {
    //在對話框中給用戶顯示標題和消息
};
Controller.prototype.success = function(msg) {
    this.showDialog("Success", msg);
};
Controller.prototype.failure = function(err) {
    this.errors.push(err);
    this.showDialog("Error", err);
};

//子類
function LoginController() {
    Controller.call(this);
}
//將子類鏈接到父類
LoginController.prototype = Object.create(Controller.prototype);
LoginController.prototype.getUser = function() {
    return document.getElementById("login_username").value;
};
LoginController.prototype.getPassword = function() {
    return document.getElementById("login_password").value;
};
LoginController.prototype.validateEntry = function(user, pw) {
    user = user || this.getUser();
    pw = pw || this.getPassword();

    if (!(user && pw)) {
        return this.failure("Please enter a username & password!");
    } else if (pw.length < 5) {
        return this.failure("Password must be 5+ characters! ");
    }

    //到這裡了？輸入合法！
    return true;
};
//覆蓋來擴展基本的 `failure()`
LoginController.prototype.failure = function(err) {
    // "super"調用
    Controller.prototype.failure.call(this, "Login invalid: " + err);
};

//子類
function AuthController(login) {
    Controller.call(this);
    //除了繼承外，我們還需要合成
    this.login = login;
}
//將子類鏈接到父類
AuthController.prototype = Object.create(Controller.prototype);
AuthController.prototype.server = function(url, data) {
    return $.ajax({
        url: url,
        data: data
    });
};
AuthController.prototype.checkAuth = function() {
    var user = this.login.getUser();
    var pw = this.login.getPassword();

    if (this.login.validateEntry(user, pw)) {
        this.server("/check-auth", {
                user: user,
                pw: pw
            })
            .then(this.success.bind(this))
            .fail(this.failure.bind(this));
    }
};
//覆蓋以擴展基本的 `success()`
AuthController.prototype.success = function() {
    // "super"調用
    Controller.prototype.success.call(this, "Authenticated!");
};
//覆蓋以擴展基本的 `failure()`
AuthController.prototype.failure = function(err) {
    // "super"調用
    Controller.prototype.failure.call(this, "Auth Failed: " + err);
};

var auth = new AuthController(
    //除了繼承，我們還需要合成
    new LoginController()
);
auth.checkAuth();
```

去類別化 :

``` JavaScript
var LoginController = {
    errors: [],
    getUser: function() {
        return document.getElementById("login_username").value;
    },
    getPassword: function() {
        return document.getElementById("login_password").value;
    },
    validateEntry: function(user, pw) {
        user = user || this.getUser();
        pw = pw || this.getPassword();

        if (!(user && pw)) {
            return this.failure("Please enter a username & password!");
        } else if (pw.length < 5) {
            return this.failure("Password must be 5+ characters! ");
        }

        //到這裡了？輸入合法！
        return true;
    },
    showDialog: function(title, msg) {
        //在對話框中向用於展示成功消息
    },
    failure: function(err) {
        this.errors.push(err);
        this.showDialog("Error ", "Login invalid: " + err);
    }
};

//鏈接 `AuthController` 委託到 `LoginController`
var AuthController = Object.create(LoginController);

AuthController.errors = [];
AuthController.checkAuth = function() {
    var user = this.getUser();
    var pw = this.getPassword();

    if (this.validateEntry(user, pw)) {
        this.server("/check-auth", {
                user: user,
                pw: pw
            })
            .then(this.accepted.bind(this))
            .fail(this.rejected.bind(this));
    }
};
AuthController.server = function(url, data) {
    return $.ajax({
        url: url,
        data: data
    });
};
AuthController.accepted = function() {
    this.showDialog("Success", "Authenticated!")
};
AuthController.rejected = function(err) {
    this.failure("Auth Failed: " + err);
};
```

## 簡化的語法

雖然在ES6出現了 class 語法，然而 OLOO 風格也是有所簡化的版本 :

``` JavaScript 
var LoginController = {

    errors: [],
    getUser() {  //看，沒有 `function` ！
        // ... 
    },
    getPassword() {
        // ... 
    }
    // ... 

}; 

``` 
不過這樣的語法，其實也有一些缺點，他所寫出來函式會變成匿名函式，也就是這個形式 :
``` JavaScript
var Foo = {
    bar: function () {  /*..*/ },
    baz: function baz() {  /*..*/ }
};
```

前面有提到三個缺陷 :

1. 使調試時的棧追踪變得困難
2. 使自引用（遞歸，事件綁定等）變得困難
3. 使代碼（稍稍）變得難於理解

1、3 在這方法並不適用，原因在於這種簡名寫法，在語言規範中有規定要去設置物件內部的 name 屬性，然而第 2 點還是無法避免，如果想要使用遞迴，還是得用具名函式。

## 型別檢查

在前面的章節中，有提到 `instanceof` 這個關鍵字，在 JS 所代表的含意與其他類別導向程式並不相同 :

``` JavaScript
function Foo() {
    /* .. */
}
Foo.prototype...

    function Bar() {
        /* .. */
    }
Bar.prototype = Object.create(Foo.prototype);

var b1 = new Bar("b1");

// `Foo` 和 `Bar` 互相的聯繫
Bar.prototype instanceof Foo; // true 
Object.getPrototypeOf(Bar.prototype) === Foo.prototype; // true 
Foo.prototype.isPrototypeOf(Bar.prototype); // true
```

其實這種繼承關係，很容易讓我們認為 `Bar instanceof Foo` 應該為 `true` ，然而事實上不是如此，必須寫到 `Bar.prototype instanceof Foo` ，原因出在 JS 一開始對於原型的設計，如果使用 OLOO 風格，問題會變得更加簡化，變成是不是原型而已。
