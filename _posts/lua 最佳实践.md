---
Title: lua 最佳实践
date: 2015-01-21 18:54
status: public
tag: cocos2d-x
title: 'lua 最佳实践'
---

关于cocos2d-x下Lua调用C++的文档看了不少，但没有一篇真正把这事给讲明白了，我自己也是个初学者，摸索了半天，总结如下：

cocos2d-x下Lua调用C++这事之所以看起来这么复杂、网上所有的文档都没讲清楚，是因为存在5个层面的知识点：

1、在纯C环境下，把C函数注册进Lua环境，理解Lua和C之间可以互相调用的本质
2、在cocos2d-x项目里，把纯C函数注册进Lua环境，理解cocos2d-x是怎样创建Lua环境的、以及怎样得到这个环境并继续自定义它 3、了解为什么要使用toLua++来注册C++类
4、在纯C++环境下，使用toLua++来把一个C++类注册进Lua环境，理解toLua++的用法 5、在cocos2d-x项目里，使用cocos2d-x注册自身的方式把自定义的C++类注册进Lua环境，理解cocos2d-x是怎样通过bindings-generator脚本来封装toLua++的用法来节省工作量的

只有理解了前4层，在最后使用bindings-generator脚本的时候心里才会清清楚楚。而网上的文档，要么是只解释了第1层，要么是只填鸭式地告诉你第5层怎么用bindings-generator脚本，不仅中间重要的知识点一概不提，示例代码往往也写的不够简洁，这让我这种看见C++就眼晕的人理解起来大为头疼（不是我不会C++，而是我非常不接受C++的设计哲学，能避就避）。所以接下来的讲解我会对每一层知识点逐一讲解，示例代码也不求完整严谨，而是尽量用最简洁的方式把程序的关键点说明白。

第一层：纯C环境下，把C函数注册进Lua环境

直接看代码比啰哩啰嗦讲一大堆概念要清晰明了的多。建立一个a.lua和一个a.c文件，内容如下，一看就明白是怎么回事了：

a.lua

print(foo(99))
a.c

#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>

int foo(lua_State *L)
{
  int n = lua_tonumber(L, 1);

  lua_pushnumber(L, n + 1);

  return 1;
}

int main()
{
  lua_State *L = lua_open();

  luaL_openlibs(L);

  lua_register(L, "foo", foo);

  luaL_dofile(L, "a.lua");

  lua_close(L);

  return 0;
}
怎么样，这代码简单吧？一看就明白，简单的不能再简单了。我特别烦示例代码里又是判断错误又是加代码注释的，本来看自己不会的代码就够吃力的了，还加那么多花花绿绿的干扰项，纯粹增加学习负担。

在命令行下用gcc来编译并执行吧：

gcc a.c -llua && ./a.out
注意-llua选项是必要的，因为要连接lua的库。

看完上面那段代码，再解释起来就容易多了：

1、要想注册进Lua环境，函数需要定义为这个样：int xxx(lua_State *L)
2、使用lua_tonumber、lua_tostring等函数，来取得传入的参数，比如lua_tonumber(L, 1)就是得到传入的第一个参数，且类型为数字
3、使用lua_pushnumber、lua_pushstring等函数，来将返回值压入Lua的环境中，因为Lua支持函数返回多个值，所以可以push多个返回值进Lua环境
4、最终函数返回的数字表示有多少个返回值被压入了Lua环境 5、使用lua_register宏定义来将这个函数注册进Lua环境，Lua脚本里就可以用它了，大功告成！就这么简单！

第二层：在cocos2d-x环境下，把C函数注册进Lua环境

也简单：

1、在frameworks/runtime-src/Classes/目录下，找到AppDelegate.cpp文件。如果frameworks目录不存在，则需要参考这篇Blog：用Cocos Code IDE写Lua，如何与项目中的C++代码和谐相处

AppDelegate.cpp文件中的关键代码如下：
```c++ auto engine = LuaEngine::getInstance();
ScriptEngineManager::getInstance()->setScriptEngine(engine);

LuaStack* stack = engine->getLuaStack();
stack->setXXTEAKeyAndSign("2dxLua", strlen("2dxLua"), "XXTEA", strlen("XXTEA"));

//register custom function
//LuaStack* stack = engine->getLuaStack();
//register_custom_function(stack->getLuaState());
可以看到cocos2d-x已经为我们留出了注册自定义C函数的位置，在注释代码后面这么写就可以了：
```cpp
lua_State *L = stack->getLuaState()；

lua_register(L, "test_lua_bind", test_lua_bind);
也可以通过ScriptEngineManager类从头取得当前的LuaEngine对象，然后再getLuaStack()方法得到封装的LuaStack对象，再调用getLuaState()得到原始的lua_State结构指针。只要知道了入口位置，其他一切就不成问题了，还是挺简单的。

感兴趣的话可以去看一下ScriptEngineManager类的详细定义，在frameworks/cocos2d-x/cocos/base/CCScriptSupport.h文件中。

BTW：这里还有一个小知识点，插入在AppDelegate.cpp中的自定义代码尽量写在COCOS2D_DEBUG宏定义的判断前面，因为在调试环境下和真机环境下后续执行的代码是不一样的：

#if (COCOS2D_DEBUG>0)
    if (startRuntime())
        return true;
#endif

    // 调试环境下代码就不会走到这里了
    engine->executeScriptFile(ConfigParser::getInstance()->getEntryFile().c_str());
    return true;
2、接下来，找个地方把test_lua_bind函数定义写进去就算大功告成了。如果追求文件组织的优雅，按理说应该新建一个.c文件，但这样的话搞不好会把自己陷入到编译阶段的泥潭里，所以先不追求优雅，而就在AppDelegate.cpp文件末尾写上函数的定义就可以了，简单清楚明了：

int test_lua_bind(lua_State *L)
{
    int number = lua_tonumber(L, 1);

    number = number + 1;

    lua_pushnumber(L, number);

    return 1;
}
3、大功告成，现在就可以在main.lua文件里使用test_lua_bind()函数了：

  local i = test_lua_bind(99)
  print("lua bind: " .. tostring(i))
4、如果是新建一个.c文件呢？把AppDelegate.cpp文件里test_lua_bind函数定义的代码删掉，在头部#include后面加入：

#include "test_lua_bind.h"
在frameworks/runtime-src/Classes目录下创建test_lua_bind.h文件，内容如下：

extern "C" {
#include "lua.h"
#include "lualib.h"
}

int test_lua_bind(lua_State *L);
再创建test_lua_bind.c文件，内容不变：

#include "test_lua_bind.h"

int test_lua_bind(lua_State *L)
{
    int number = lua_tonumber(L, 1);

    number = number + 1;

    lua_pushnumber(L, number);

    return 1;
}
此时用cocos compile -p mac命令编译，会发现test_lua_bind.c文件并没有被编译。这是当然的，普通的C/C++项目都是用Makefile来指定编译哪些.c/cpp文件的，当前的cocos2d-x项目虽然没有Makefile文件，但也是遵循这个原则的，也即肯定是有一个地方来指定所有要编译的文件的，需要在这个地方把test_lua_bind.c加进去，使得整个项目编译时把它也作为项目的一部分。

答案是，cocos2d-x项目没有使用Makefile，而是非常聪明地使用了与具体环境相关的工程文件来作为命令行编译的环境，比如在编译iOS或Mac时就使用Xcode工程文件，在编译Android时就使用Android.mk文件。

所以，添加好了test_lua_bind.h和test_lua_bind.c文件后，用Xcode打开项目，将这俩文件添加进工程中就行了。

Xcode中添加.h和.cpp文件进工程

注意，千万不要勾选“Copy items into destination group's folder(if needed)”，因为cocos2d-x的Xcode工程目录组织不是常规的结构，一旦勾选这个，会导致这两个文件被拷贝至frameworks/runtime-src/proj.ios_mac目录下，原来frameworks/runtime-src/Classes目录下的文件就废掉了，这样的组织方式会乱，而且会影响Android那边对这俩文件的引用。

把test_lua_bind.h和test_lua_bind.cpp这俩文件添加进Xcode工程后，再去命令行执行cocos compile -p mac，编译就能成功了。

网上有其他文章说还要修改Xcode工程的“User Headers Path”，这个经过试验是不需要的，哪怕把这俩文件放进新建的文件夹里也不需要，只要加入了Xcode工程即可，因为Xcode内部根本就不是按照文件夹的形式来组织文件的，它自己有一套叫做“Group”的东西。搞了好几年iOS开发，对Xcode的这个特性还是熟悉的。

Xcode工程中添加了test_lua_bind.h/cpp文件后的样子

说到这就不禁要插一句对网上所有cocos2d-x文档的吐槽了，学习cocos2d-x的人水平实在是良莠不齐，大部分人似乎都是对游戏热衷的编程初学者，他们大多底子薄基础差，甚至一大部分人之前都没做过移动APP的开发，他们学习cocos2d-x只想知其然而不想知其所以然，给他们讲他们也看不明白（因为编程基础差），所以网上不少cocos2d-x文章都是只讲123步骤，而不告诉你为什么这么做，包括cocos2d-x官方的大量文档也是基于这个思路写的，中文和英文都一样。我看这些文章就特别痛苦，一边看一边心里就总是在想，“凭什么要这么做啊”、“这一步是为了什么啊”、“怎么这么麻烦啊”、“这个步骤明显不是最佳实践啊”、“解决这事为啥要这么麻烦”、“有更好的方法吗”，所以我这种初学者来看cocos2d-x文档就变成了不是单纯的学习，而是学习、质疑、求证、反思、优化的过程，对别人来说cocos2d-x的入门比较容易，到我这里反倒成了入门比较难、入门之后比较容易了，因为文档中的垃圾信息和无效信息实在是太多了，别人可以照单全收、以后懂了之后再慢慢剔除，我是必须从一开始就自己甄别垃圾、只保留最佳实践，这也是这篇Blog写的比较长的原因。

扯远了。反正经过以上步骤，就完成了在cocos2d-x项目中把C函数注册进Lua环境这件事。至此，算是彻底搞懂了Lua和C函数之间的互相调用关系，也能在cocos2d-x的Lua环境中使用自定义的C函数了。但这还不够，因为一个正规的项目是需要狠好的组织结构的，全局C函数满天飞肯定是不行的，好一点的情况是把所有的C函数都在Lua中组织为模块注册进去，更好一点的情况是把C++类注册进Lua、并且C++类也是以Lua模块为组织方式注册进Lua环境的。这其实就是cocos2d-x自己把自己注册进Lua环境的方式。

第三层：了解为什么要使用toLua++来注册C++类

因为Lua的本质是C，不是C++，Lua提供给C用的API也都是基于面向过程的C函数来用的，要把C++类注册进Lua形成一个一个的table环境是不太容易一下子办到的事，因为这需要绕着弯地把C++类变成各种其他类型注册进Lua，相当于用面向过程的思维来维护一个面向对象的环境。这其中的细节就不去深究了，总之正是因为如此，所以单纯地手写lua_register()等代码来注册C++类是行不通的、代价高昂的，所以需要借助toLua++这个工具。

这一层的知识点看似简单，但其实是非常重要的，只有理解了手工用lua_register()去注册C++类的难度，才能理解使用toLua++这类工具的必要性。只有理解了使用toLua++工具的必要性，才会潜下心来冷静地接受toLua++本身的优点和缺点。只有看到了toLua++本身的缺点和使用上的麻烦，才会真心理解cocos2d-x使用bindings-generator脚本带来的好处。只有理解了bindings-generator脚本带来的好处，才能谅解这个脚本本身在使用上的一些不便之处。

第四层：在纯C++环境下，使用toLua++来把一个C++类注册进Lua环境

虽然终极方法是用bindings-generator脚本来注册C++类进cocos2d-x的Lua环境，但理解toLua++本身的用法还是狠有必要的，只有知道了toLua++原本的用法，才能更好地理解cocos2d-x是怎么把自己的C++类都注册进Lua环境的，这不仅能让编程时的思路更加清晰，也能为日后在源码中寻找各种接口文档的过程中不至于看不懂那一大堆tolua_beginmodule、tolua_function是什么意思。影响程序员学习提高的一大障碍就是忽略那些一知半解的代码，不去刨根究底地搞明白。

使用toLua++的标准做法是：

1、准备好自己的C++类，该怎么写就怎么写
2、仿造这个类的.h文件，改一个.pkg文件出来，具体格式要按照toLua++的规定，比如移除所有的private成员等 3、建一个专门用来桥接C++和Lua之间的C++类，使用特殊的函数签名来写它的.h文件，.cpp文件不写，等着toLua++来生成
4、给这个桥接的C++类写一个.pkg文件，按照toLua++的特殊格式来写，目的是把真正做事的C++类给定义进去 5、在命令行下用toLua++生成桥接类的.cpp文件
6、程序入口引用这个桥接类，执行生成的桥接函数，Lua环境中就可以使用真正做事的C++类了

toLua++这种自己手写.pkg文件的方式古老又难受，所以我没有仔细地去学习，这套流程放在10年前的那个年代是没有太大问题的，作者怎么规定就怎么用好了，但是放在2014年的今天，任何程序的架构设计都讲究学习成本低、轻量化、符合以往的习惯，因此toLua++用起来我觉得其实是难受的。

下面我以尽量最少的代码来走一遍toLua++的流程，注意这是在纯C++环境下，跟任何框架都没关系，也不考虑内存释放等细节：

MyClass.h

class MyClass {
public:
  MyClass() {};

  int foo(int i);
};
MyClass.cpp

#include "MyClass.h"

int MyClass::foo(int i)
{
  return i + 100;
}
MyClass.pkg

class MyClass
{
  MyClass();
  int foo(int i);
};
MyLuaModule.h

extern "C" {
#include "tolua++.h"
}

#include "MyClass.h"

TOLUA_API int tolua_MyLuaModule_open(lua_State* tolua_S);
MyLuaModule.pkg

$#include "MyLuaModule.h"

$pfile "MyClass.pkg"
main.cpp

extern "C" { 
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
}

#include "MyLuaModule.h"

int main()
{
  lua_State *L = lua_open();

  luaL_openlibs(L);

  tolua_MyLuaModule_open(L);

  luaL_dofile(L, "main.lua");

  lua_close(L);

  return 0;
}
main.lua

local test = MyClass:new()
print(test:foo(99))
先在命令行下执行：

tolua++ -o MyLuaModule.cpp MyLuaModule.pkg
此命令用来生成桥接文件MyLuaModule.cpp。注意命令行中-o参数的顺序不能随意摆放，从这个小事也能看出tolua++的古老和难用

生成好MyLuaModule.cpp文件后，就能看到它里面的那一大堆桥接代码了，比如tolua_beginmodule、tolua_function等。以后看到这些东西就不陌生了，就明白这些函数只是toLua++用来做桥接的必备代码了，简单看一下代码，就理解toLua++是怎样把MyClass这个C++类注册进Lua中的了：

toLua++生成的桥接代码

接下来，用g++来编译：

g++ MyClass.cpp MyLuaModule.cpp main.cpp -llua -ltolua++
默认就生成了a.out文件，执行，就能看到main.lua的执行结果了：

toLua++的执行结果

至此，对toLua++的运作原理心里就透亮了，无非就是：

1、把自己该写的类写好
2、写个.pkg文件，告诉toLua++这个类暴露出哪些接口给Lua环境 3、再写个桥接的.h和.pkg文件，让toLua++去生成桥接代码
4、在程序里使用这个桥接代码，类就注册进Lua环境里了

第五层：使用cocos2d-x的方式来将C++类注册进Lua环境

cocos2d-x在2.x版本里就是用toLua++和.pkg文件这么把自己注册进Lua环境里的。不过这种方法明显笨拙，既要写真正做事的.pkg文件，也要写桥接的.pkg文件和.h文件，工作量又大又枯燥。所以从cocos2d-x 3.x开始，用bindings-generator脚本代替了toLua++。

bindings-generator脚本的工作机制是：

1、不用挨个类地写桥接.pkg和.h文件了，直接定义一个ini文件，告诉脚本哪些类的哪些方法要暴露出来，注册到Lua环境里的模块名是什么，就行了，等于将原来的每个类乘以3个文件的工作量变成了所有类只需要1个.ini文件
2、摸清了toLua++工具的生成方法，改由Python脚本动态分析C++类，自动生成桥接的.h和.cpp代码，不调用tolua++命令了 3、虽然不再调用tolua++命令了，但是底层仍然使用toLua++的库函数，比如tolua_function，bindings-generator脚本生成的代码就跟使用toLua++工具生成的几乎一样

bindings-generator脚本掌握了生成toLua++桥接代码的主动权，不仅可以省下大量的.pkg和.h文件，而且可以更好地插入自定义代码，达到cocos2d-x环境下的一些特殊目的，比如内存回收之类的。所以cocos2d-x从3.x开始放弃了toLua++和.pkg而改用了自己写的bindings-generator脚本是非常值得赞赏的聪明做法。

接下来说怎么用bindings-generator脚本：

1、写自己的C++类，按照cocos2d-x的规矩，继承cocos2d::Ref类，以便使用cocos2d-x的内存回收机制。当然不这么干也行，但是不推荐，不然在Lua环境下对象的释放狠麻烦。
2、编写一个.ini文件，让bindings-generator可以根据这个配置文件知道C++类该怎么暴露出来 3、修改bindings-generator脚本，让它去读取这个.ini文件
4、执行bindings-generator脚本，生成桥接C++类方法 5、用Xcode将自定义的C++类和生成的桥接文件加入工程，不然编译不到
6、修改AppDelegate.cpp，执行桥接方法，自定义的C++类就注册进Lua环境里了

看着步骤挺多，其实都狠简单。下面一步一步来。

首先是自定义的C++类。我习惯将文件保存在frameworks/runtime-src/Classes/目录下：

frameworks/runtime-src/Classes/MyClass.h

#include "cocos2d.h"

using namespace cocos2d;

class MyClass : public Ref
{
public:
  MyClass()   {};
  ~MyClass()  {};
  bool init() { return true; };
  CREATE_FUNC(MyClass);

  int foo(int i);
};
frameworks/runtime-src/Classes/MyClass.cpp

#include "MyClass.h"

int MyClass::foo(int i)
{
  return i + 100;
}
然后编写.ini文件。在frameworks/cocos2d-x/tools/tolua/目录下能看到genbindings.py脚本和一大堆.ini文件，这些就是bindings-generator的实际执行环境了。随便找一个内容比较少的.ini文件，复制一份，重新命名为MyClass.ini。大部分内容都可以凑合不需要改，这里仅列出必须要改的重要部分：

frameworks/cocos2d-x/tools/tolua/MyClass.ini

[MyClass]
prefix           = MyClass
target_namespace = my
headers          = %(cocosdir)s/../runtime-src/Classes/MyClass.h
classes          = MyClass
也即在MyClass.ini中指定MyClass.h文件的位置，指定要暴露出来的类，指定注册进Lua环境的模块名。

注意，这个地方我踩了个坑。如果.ini配置文件中存在macro_judgement = ...宏定义，要特别小心，我第一次是从cocos2dx_controller.ini文件复制来的，结果没注意macro_judgement，导致生成的桥接类文件加入了不该加入的宏，只在iOS和Android平台上才起作用，对Mac平台无效，这个要特别注意。

然后修改genbindings.py文件129行附近，将MyClass.ini文件加进去：

frameworks/cocos2d-x/tools/tolua/genbindings.py

cmd_args = {'cocos2dx.ini' : ('cocos2d-x', 'lua_cocos2dx_auto'), \
            'MyClass.ini' : ('MyClass', 'lua_MyClass_auto'), \
            ...
（其实这一步本来是可以省略的，只要让genbindings.py脚本自动搜寻当前目录下的所有ini文件就行了，不知道将来cocos2d-x团队会不会这样优化）

至此，生成桥接文件的准备工作就做好了，执行genbindings.py脚本：

python genbindings.py
（在Mac系统上可能会遇到缺少yaml、Cheetah包的问题，安装这些Python包狠简单，先sudo easy_install pip，把pip装好，然后用pip各种pip search、sudo pip install就可以了）

成功执行genbindings.py脚本后，会在frameworks/cocos2d-x/cocos/scripting/lua-bindings/auto/目录下看到新生成的文件：

成功执行genbindings.py后生成的桥接C++文件

每次执行genbindings.py脚本时间都挺长的，因为它要重新处理一遍所有的.ini文件，建议大胆修改脚本文件，灵活处理，让它每次只处理需要的.ini文件就可以了，比如像这个样子：

修改genbindings.py使其只生成自定义的桥接类

在frameworks/cocos2d-x/cocos/scripting/lua-bindings/auto/目录下观察一下生成的C++桥接文件lua_MyClass_auto.cpp，里面的注册函数名字为register_all_MyClass()，这就是将MyClass类注册进Lua环境的关键函数：

生成的桥接文件内容

编辑frameworks/runtime-src/Classes/AppDelegate.cpp文件，首先在文件头加入对lua_MyClass_auto.hpp文件的引用：

AppDelegate.cpp文件的头加入对lua_MyClass_auto.hpp文件的引用

然后在正确的代码位置加入对register_all_MyClass函数的调用：

修改AppDelegate.cpp文件，将MyClass类注册进Lua环境

最后在执行编译前，将新加入的这几个C++文件都加入到Xcode工程中，使得编译环境知道它们的存在：

在Xcode中加入新冒出来的C++文件

这其中还有一个小坑，由于lua_MyClass_auto.cpp文件要引用MyClass.h文件，而这俩文件分属于不同的子项目，互相不认识头文件的搜寻路径，因此需要手工修改一下cocos2d_lua_bindings.xcodeproj子项目的User Header Search Paths配置。特别注意一共有几个../：

需要修改cocos2d_lua_bindings子项目的User Header Search Paths配置

最后，就可以用cocos compile -p mac命令重新编译整个项目了，不出意外的话编译一定是成功的。

修改main.lua文件中，尝试调用一下MyClass类：

local test = my.MyClass:create()
print("lua bind: " .. test:foo(99))
然后执行程序（用cocos rum -p mac或在Cocos Code IDE中均可），见证奇迹的时刻~~~~咦我擦？！程序崩溃！为毛？

第一次运行绑定了C++类的程序居然崩溃

这是我作为cocos2d-x初学者遇到的最大的坑，坑了我整整一天半，具体的研究细节就不详细说了，总之罪魁祸首是cocos2d-x框架中的CCLuaEngine.cpp文件的这段代码：

CCLuaEngine.cpp执行的这段代码要了命了

原因是executeScriptFile函数执行时，对当前Lua环境中的栈进行了清理，当register_all_MyClass函数被调用时，Lua栈是全空的状态，函数内部执行到tolua_module函数调用时就崩溃了：

CCLuaEngine.cpp中对executeScriptFile方法的定义清理了Lua栈

解决办法是修改AppDelegate.cpp为这个样子：

AppDelegate.cpp调用register_all_MyClass函数时要重新恢复一下栈的内容才行

文本形式的代码如下：

AppDelegate.cpp

lua_State *L = stack->getLuaState();
lua_getglobal(L, "_G");
register_all_MyClass(L);
lua_settop(L, 0);
重新编译并执行，程序就正确执行了：

程序终于正确执行了

至此，就彻底搞清楚应该怎样在cocos2d-x项目里绑定一个C函数或者C++类到Lua环境中了，感兴趣的话可以再进一步深入研究Lua内部metatable的运作原理、类对象的生成与释放、以及垃圾回收。我自己也是刚接触cocos2d-x不到一个星期，理解不深，以上难免会有用词不当或理解错误的地方，如有错误请多包涵。

后记补充：如果C++类定义了namespace，则需要修改frameworks/cocos2d-x/tools/bindings-generator/targets/lua/conversions.yaml文件，定义namespace与Lua之间的映射关系，否则会报conversion wasn't set错误：

C++的类如果定义了namespace，则需要在导出Lua时修改conversions.yaml文件