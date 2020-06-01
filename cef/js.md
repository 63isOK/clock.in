# js/c++交互

## 介绍

cef内部使用v8引擎来处理js,
对于browser进程上的每一个frame,都有各自的js上下文,
这个frame中在js上下文中提供了安全和作用域的属性.

cef3中,js的执行是放在render进程中的,
render进程的主线程,标记为"TID_RENDERER",
v8就是在主线程中执行.
js执行的回调,是通过CefRenderProcessHandler接口来暴露的,
renderer进程初始化后,这个接口是和CefApp::GetRenderProcessHandler()
就绑定了.

分析一下:js在browser上,对js解析的v8在renderer上,
在browser和renderer之间交互的js api都被设计成"异步回调".

## 客户端执行js

c++中执行js使用CefFrame::ExecuteJavaScript(),
这个函数可以在非js上下文的browser/renderer中安全调用,
为什么前缀是"非js上下文",因为如果在js上下文中,
是可以直接调用js的,所以下面将讨论范围限定在非js上下中.

    // 在非js上下文中执行js代码
    CefRefPtr<CefFrame> frame = browser->GetMainFrame();
    frame->ExecuteJavaScript("alert('ExecuteJavaScript works!');",
     frame->GetURL(), 0}
    // 我们在第一个browser的主frame中执行了js代码

这个函数除了可以执行函数,也可以设置变量,
但为了处理js执行的返回值,那就需要用到window binding或extensions.

## window绑定 window binding

window绑定允许client app将一个值附加到frame的window对象中.
实现是通过CefRenderProcessHandler::OncontextCreated()方法.

    void MyRenderProcessHandler::OnContextCreated(
        CefRefPtr<CefBrowser> browser,
        CefRefPtr<CefFrame> frame,
        CefRefPtr<CefV8Context> context) {

      CefRefPtr<CefV8Value> object = context->GetGlobal();
      CefRefPtr<CefV8Value> str = CefV8Value::CreateString("My Value!");
      object->SetValue("myval", str, V8_PROPERTY_ATTRIBUTE_NONE);
    }
    // 可以看到,在renderer进程中,可以修改frame的js上下文

window绑定的好处是:每次frame刷新时,都有机会去修改js上下文.

通过这种方式,是可以读写js上下文的.

## extensions

姑且称extensions为扩展,扩展相比window绑定,扩展不会在frame每次刷新都改变,
当扩展去修改dom时,如果dom还未创建,那么会导致崩溃.

扩展的注册由CefRegisterExtension()完成,
注册函数在CefRenderProcessHandler::OnWebKitInitialized()方法中来调用.

通过这种方式,每个frame都会加载注册的扩展,且不会被frame的刷新而修改.

这种读写js上下文的方式是对window绑定的补充.

## 基本的js类型

cef支持的js类型如下:

- 未定义
- null
- bool
- int
- double
- date
- string

通过CefV8Value::Createxxx()静态方法来创建.
任何时候都可以创建基本js类型,创建时不需要和某个具体的js上下文绑定.

通过Isxxx()方法来判断值得类型;通过GetxxxValue()来获取值.

    CefRefPtr<CefV8Value> str = CefV8Value::CreateString("My Value!");
    CefRefPtr<CefV8Value> val = ...;
    if (val.IsString()) {
      // The value is a string.
    }
    CefString strVal = val.GetStringValue();

## js数组

CefV8Value::CreateArray()会创建一个js数组,长度又参数决定.
和基本js类型不一样的是:js数组只能在js上下文中创建和使用.

赋值用SetValue(),类型判断用IsArray(),
GetArrayLength()/GetValue()都是常用操作.

    // Create an array that can contain two values.
    CefRefPtr<CefV8Value> arr = CefV8Value::CreateArray(2);
    // Add two values to the array.
    arr->SetValue(0, CefV8Value::CreateString("My First String!"));
    arr->SetValue(1, CefV8Value::CreateString("My Second String!"));

## js对象

CefV8Value::CreateObject()创建js对象,可选参数CefV8Accessor.
js对象也只能在js上下文中创建和使用.

SetValue()是设置值,会带一个key.

    CefRefPtr<CefV8Value> obj = CefV8Value::CreateObject(NULL);
    obj->SetValue("myval", CefV8Value::CreateString("My String!"));

上面例子中,创建js对象时,参数是NULL.
参数也可以是对象访问器,由对象访问器来实现读写.
js对象访问器具体可以查看cef官方文档.

## js函数

CefV8Value::CreateFunction()可以创建js函数,
js函数同样只能在js上下文中创建和使用.

我们在C++端创建了一个js函数,可以在js代码中调用,
结合window绑定和扩展,就有两种用法.

定义一个js函数:

    CefRefPtr<CefV8Handler> handler = …;
    CefRefPtr<CefV8Value> func = CefV8Value::CreateFunction("myfunc", handler);

    class MyV8Handler : public CefV8Handler {
    public:
      MyV8Handler() {}

      virtual bool Execute(const CefString& name,
                           CefRefPtr<CefV8Value> object,
                           const CefV8ValueList& arguments,
                           CefRefPtr<CefV8Value>& retval,
                           CefString& exception) OVERRIDE {
        if (name == "myfunc") {
          // Return my string value.
          retval = CefV8Value::CreateString("My Value!");
          return true;
        }

        // Function does not exist.
        return false;
      }

      // Provide the reference counting implementation for this class.
      IMPLEMENT_REFCOUNTING(MyV8Handler);
    };

在window绑定中使用js函数:

    void MyRenderProcessHandler::OnContextCreated(
        CefRefPtr<CefBrowser> browser,
        CefRefPtr<CefFrame> frame,
        CefRefPtr<CefV8Context> context) {
      // Retrieve the context's window object.
      CefRefPtr<CefV8Value> object = context->GetGlobal();

      // Create an instance of my CefV8Handler object.
      CefRefPtr<CefV8Handler> handler = new MyV8Handler();

      // Create the "myfunc" function.
      CefRefPtr<CefV8Value> func = CefV8Value::CreateFunction("myfunc", handler);

      // Add the "myfunc" function to the "window" object.
      object->SetValue("myfunc", func, V8_PROPERTY_ATTRIBUTE_NONE);
    }

    // 在js中的调用如下:
    <script language="JavaScript">
    alert(window.myfunc()); // Shows an alert box with "My Value!"
    </script>

在扩展中使用js函数,需要注意的是:需要添加"native function"前置说明.

    void MyRenderProcessHandler::OnWebKitInitialized() {
      // Define the extension contents.
      std::string extensionCode =
        "var test;"
        "if (!test)"
        "  test = {};"
        "(function() {"
        "  test.myfunc = function() {"
        "    native function myfunc();"
        "    return myfunc();"
        "  };"
        "})();";

      // Create an instance of my CefV8Handler object.
      CefRefPtr<CefV8Handler> handler = new MyV8Handler();

      // Register the extension.
      CefRegisterExtension("v8/test", extensionCode, handler);
    }

    // 在js中调用
    <script language="JavaScript">
    alert(test.myfunc()); // Shows an alert box with "My Value!"
    </script>

## 上下文

每个frame都有一个v8上下文.frame包含了变量/函数/对象的作用域.
如果代码中有CefV8Handler/CefV8Acessor/OnContextCreated/OnContextReleased()
等级别更高的回调,那这个上下文就包含V8.

和frame相关的V8上下文的生命周期,由OnContextCreated()/OnContextReleased()决定,使用这两个函数时,需要注意以下规则:

- 调用OnContextReleased()之后,不能保留或引用V8上下文
- V8对象的生命周期是不定的(实际上是取决于GC),所以引用V8对象要格外注意,最好使用代理对象

如果是跨frame调用函数,此时存在切换上下文,
这种特殊场景要查看官方文档.
GetCurrentContext()/GetEnteredContext()会有帮助.

数组/对象/函数,他们的创建修改调用都在js上下文中,
如果此时V8不在js上下文中,需要调用Enter()/Exit().
具体这块,也需要查看官方文档.

## 执行js函数

前面一直在聊"在C++中如何构建js元素,以便js代码可以调用这些元素".
本节聊一下"在C++中如何调用js函数".

ExecuteFunction()/ExecuteFunctionWithContext()函数会帮上忙.
ExecuteFunction()的使用场景:V8上下文已经包含在js上下文中,
ExecuteFunctionWithContext()可以指定上下文.

### 使用js回调

上面指出了调用js函数的两个函数,这部分讨论如何调用js的函数.

在C++代码中,需要保存:上下文和js函数的引用.

关于js值的读写,前面提供了两种方式:window绑定和扩展,
下面着重说一下通过window绑定来实现调用js函数.

- 在window绑定中,创建一个register函数

可以不是register函数,可以是任意想注册的函数.

    void MyRenderProcessHandler::OnContextCreated(
        CefRefPtr<CefBrowser> browser,
        CefRefPtr<CefFrame> frame,
        CefRefPtr<CefV8Context> context) {
      // Retrieve the context's window object.
      CefRefPtr<CefV8Value> object = context->GetGlobal();

      CefRefPtr<CefV8Handler> handler = new MyV8Handler(this);
      object->SetValue("register",
                       CefV8Value::CreateFunction("register", handler),
                       V8_PROPERTY_ATTRIBUTE_NONE);
    }

可以看出,这是一个典型的window绑定过程,
给window对象绑定的是一个函数,这里没有写明函数的具体内容.

- 在MyV8Handler::Execute()中实现函数的具体内容

在这一步,需要保存js函数和上下文的引用.

    bool MyV8Handler::Execute(const CefString& name,
                              CefRefPtr<CefV8Value> object,
                              const CefV8ValueList& arguments,
                              CefRefPtr<CefV8Value>& retval,
                              CefString& exception) {
      if (name == "register") {
        if (arguments.size() == 1 && arguments[0]->IsFunction()) {
          callback_func_ = arguments[0];
          callback_context_ = CefV8Context::GetCurrentContext();
          return true;
        }
      }

      return false;
    }

在这里,如果在html中调用window.函数(),会走到上面的函数里,
这个注册函数做的东西其实很简单:保存上下文和第一个参数,
第一个参数其实就是js函数,就是我们要调用的.

- 在js代码中,注册js回调

在js代码中,确实将实际的js函数作为注册函数的第一个参数.

    <script language="JavaScript">
    function myFunc() {
      // do something in JS.
    }
    window.register(myFunc);
    </script>

- 最后一步,调用

此时已经获取了js上下文和js函数的引用,
可以用上面说到的两个函数来执行了.

    CefV8ValueList args;
    CefRefPtr<CefV8Value> retval;
    CefRefPtr<CefV8Exception> exception;
    if (callback_func_->ExecuteFunctionWithContext(
      callback_context_, NULL, args, retval, exception, false)) {
      if (exception.get()) {
        // Execution threw an exception.
      } else {
        // Execution succeeded.
      }
    }

### 抛出异常

在CefV8Value::ExecuteFunctionxxx()调用之前,
如果先调用了CefV8Value::SetRethrowExceptions(true),
那V8执行期间出现的任意异常都会被重新抛出.
只要有异常重新抛出,任何C++代码需要立马return.
只有js调用的级别比堆栈高时,异常才能重新抛出.

关于异常重新抛出,可查看官方文档.
