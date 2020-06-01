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
