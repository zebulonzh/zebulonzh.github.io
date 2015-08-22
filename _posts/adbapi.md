---
Title: adbapi
date: 2015-01-15 21:02
status: draft
tag: Server
---

#1. Scheduling tasks for the future
> 要在未来X秒的时候运行一个任务，这种方式定义在reactor接口里面
``` python
twisted.internet.interfaces.IReactorTime:

from twisted.internet import reactor
def f(s):
    print "this will run 3.5 seconds after it was scheduled: %s" % s
reactor.callLater(3.5, f, "hello, world")
# f() will only be called if the event loop is started.
reactor.run()
```
> 如果函数结果很重要或者有必要处理函数抛出的异常，那么twisted.internet.task.deferLater功能可以很方便的创建一个Deferred并设置一个延迟调用： 
``` python
from twisted.internet import task
from twisted.internet import reactor
def f(s):
  return "This will run 3.5 seconds after it was scheduled: %s" % s
d = task.deferLater(reactor, 3.5, f, "hello, world")
def called(result):
  print result
d.addCallback(called)
# f() will only be called if the event loop is started.
reactor.run()
```
> 如果我们要让一个任务在未来每X秒运行一次，我们可以用twisted.internet.task.LoopingCall: 
``` python
from twisted.internet import task
from twisted.internet import reactor
def runEverySecond():
    print "a second has passed"
l = task.LoopingCall(runEverySecond)
l.start(1.0) # call every second
# l.stop() will stop the looping calls
reactor.run()
```
> 如果我们要取消一个已经在调度(we have scheduled)的任务（应该是指已经加入调度队列的任务）： 
``` python
from twisted.internet import reactor
def f():
    print "I'll never run."
callID = reactor.callLater(5, f)
callID.cancel()
reactor.run()
```
> 就像所有基于reactor的代码(reactor-based code)一样，为了让调度运行必须启动reactor: reactor.run().
 
--------
#2. twisted.enterprise.adbapi: Twisted RDBMS support

> Twsited是一个异步网络框架，但是不幸的是大部分数据库API的实现都是阻塞接口---鉴于这个原因，twisted.enterprise.adbapi横空出世。它是针对标准 DB-API 2.0 API 的非阻塞接口，而且可以访问许多不同的RDBMSes.
enterpreise.adbapi会在单独的线程里面执行阻塞的数据库操作，当数据库操作完成的时候又会在原来的线程(originating thread，可能是说twisted的主事件循环线程)中触发回调。在此期间，原来的线程可以继续做正常的工作，比如服务于其他请求。
我们并非要直接创建数据库连接，而是用adbapi.ConnectionPool类管理数据库连接。这东东允许enterprise.adbapi使用多个数据库连接，每个线程一个。很简单:
``` python
# Using the "dbmodule" from the previous example, create a ConnectionPool 
from twisted.enterprise import adbapi 
dbpool = adbapi.ConnectionPool("dbmodule", 'mydb', 'andrew', 'password')
```
>需要注意的是：
不需要直接import dbmodule。只用把module的名字传递给adbapi.ConnectionPool的构造函数就行了。
需要传递给dbmodule.connect()的参数应当做额外的参数传递给adbapi.ConnectionPool的构造函数。关键字参数也没问题。
现在我们来做一个数据库查询：
``` python
# equivalent of cursor.execute(statement), return cursor.fetchall():
def getAge(user):
  return dbpool.runQuery("SELECT age FROM users WHERE name = ?", user)
def printResult(l):
  if l:
    print l[0][0], "years old"
  else:
    print "No such user"
```
> 除了getAge的返回值以外，其他都很直观。getAge返回了一个twisted.internet.defer.Deferred,这个Deferred允许在数据库操作完成的时候对于结果或者错误执行任意的callback。
除了runQuery,还有runOperation和runInteraction可以被调用。The function will be called in the thread with a twisted.enterprise.adbapi.Transaction, which basically mimics a DB-API cursor. 在所有情况下，一个数据库的事务会在这次数据库使用结束的时候提交，除非抛出了异常（如果抛出了异常就会回滚）。
``` python
def _getAge(txn, user):
  # this will run in a thread, we can use blocking calls
  txn.execute("SELECT * FROM foo")
  # ... other cursor commands called on txn ...
  txn.execute("SELECT age FROM users WHERE name = ?", user)
  result = txn.fetchall()
  if result:
    return result[0][0]
  else:
    return None
def getAge(user):
  return dbpool.runInteraction(_getAge, user)
def printResult(age):
  if age != None:
    print age, "years old"
  else:
    print "No such user"
getAge("joe").addCallback(printResult)
```
> 这里都假定了dbmodule使用"qmark"风格的参数(parmstyle)(参见DB-API说明书)，这是不碍事的。如果我们的dbmodule要使用一种不同的参数风格(比如pyformat),那么直接用就是了。twisted没有试图提供任何类型的魔法参数转换---runQuery(query,aprams,...)直接映射到cursor.execute(query,params,...).
Examples of various database adapters 
注意，第一个参数是我们要import和使用connect(...)的module的名字，接下来的参数是我们调用connect(...)的参数。
``` python
from twisted.enterprise import adbapi
# Gadfly
cp = adbapi.ConnectionPool("gadfly", "test", "/tmp/gadflyDB")
# PostgreSQL PyPgSQL
cp = adbapi.ConnectionPool("pyPgSQL.PgSQL", database="test")
# MySQL
cp = adbapi.ConnectionPool("MySQLdb", db="test")
```
===================================================================================================================================================================== 
#3. Using Threads in Twisted & Running code in a thread-safe manner 
> Twisted中的大部分代码非线程安全。例如，往一个protocol的transport写数据就不是线程安全的。所以，我们需要一种方式来把函数调度到主事件循环里面去运行。这个可用函数twisted.internet.interfaces.IReactorThreads.callFromThread来做到：

``` python
from twisted.internet import reactor
def notThreadSafe(x):
   """do something that isn't thread-safe"""
   # ...
def threadSafeScheduler():
  """Run in thread-safe manner."""
  reactor.callFromThread(notThreadSafe, 3) # will run 'notThreadSafe(3)'
                       # in the event loop
```
#4. Running code in threads 
> 有时我们需要在线程中运行函数--例如，要访问阻塞的API。twisted用IReactorThreads API(twisted.internet.interfaces.IReactorThreads)提供这种方法。其他的功能性函数在twisted.internet.threads中提供。基本上，这些方法可以让我们把函数排进队列里让线程池运行了。
例如，要在一个线程中运行一个函数我们可以这样做：

``` python
from twisted.internet import reactor
def aSillyBlockingMethod(x):
  import time
  time.sleep(2)
  print x
# run method in thread
reactor.callInThread(aSillyBlockingMethod, "2 seconds have passed")
reactor.run()
```
> 功能性函数（utility methods）不是twisted.internet.reactor APIs的一部分，而是在twisted.internet.threads中实现。
如果我们有多个函数要在一个线程中序列化运行，我们可以这么做：
``` python
from twisted.internet import reactor, threads
def aSillyBlockingMethodOne(x):
  import time
  time.sleep(2)
  print x
def aSillyBlockingMethodTwo(x):
  print x
# run both methods sequentially in a thread
commands = [(aSillyBlockingMethodOne, ["Calling First"], {})]
commands.append((aSillyBlockingMethodTwo, ["And the second"], {}))
threads.callMultipleInThread(commands)
reactor.run()
```
> 对于我们需要获取其结果的函数，我们可以让结果作为一个Deferred返回：
``` python
from twisted.internet import reactor, threads
def doLongCalculation():
  # .... do long calculation here ...
  return 3
def printResult(x):
  print x
# run method in thread and get result as defer.Deferred
d = threads.deferToThread(doLongCalculation)
d.addCallback(printResult)
reactor.run()
```
> 如果我们要在reactor线程中调用一个函数并获取它的结果，我们可以用blockingCallFromThread:
``` python
from twisted.internet import threads, reactor, defer
from twisted.web.client import getPage
from twisted.web.error import Error
def inThread():
  try:
    result = threads.blockingCallFromThread(
      reactor, getPage, "http://twistedmatrix.com/")
  except Error, exc:
    print exc
  else:
    print result
  reactor.callFromThread(reactor.stop)
reactor.callInThread(inThread)
reactor.run()
```
> blockingCallFromThread将把传递给它的函数所返回的对象返回来、或者把传递给它的函数raise的异常raise出来。如果传给它的函数返回一个Deferred,那么它会返回Deferred的callback收到的数据或者raise Deferred的errback收到的异常（ If the function passed to it returns a Deferred, it will return the value the Deferred is called back with or raise the exception it is errbacked with. ）
Managing the Thread Pool 
线程池由twisted.python.threadpool.ThreadPool实现。

我们可能想修改threadpool的大小，增加或减少使用的线程的数量。我们可以很容易的做到：

from twisted.internet import reactor
reactor.suggestThreadPoolSize(30)
线程池的默认大小取决于所使用的reactor；默认的reactor使用的线程池最小线程数为5最大线程数为10.在修改线程池大小之前一定要小心并且要保证已经理解线程以及线程对资源的使用。
===================================================================================================================================================================== 
Deferred Reference 
Defrereds 
twisted用Deferred对象管理回调序列。客户程序把一系列的函数附加到deferred上，当异步请求的结果可用时这些函数会被调用,这一系列的函数就是回调(callback)，或者说回调链(callback chain).如果异步请求发生错误也会有一系列的函数被调用，这些函数就是错误回调(errback)或者错误回调链(errback chain)。当结果可用时异步库的代码调用第一个callback，或者当错误发生时调用第一个errback， Deferred对象会把每个callback/errback函数的结果hold住并传入链中的下一个函数 。

twisted.internet.defer.Deferred对象是函数将会在某个点上产生结果的承诺。我们可以把callback函数附加到给一个Deferred，一旦Deferred得到了结果这些callbacks就会被执行。

如果不显式的return a twisted.python.failure.Failure或者raise an exception，错误会停止传播 ，正常的callbacks会从这个点上继续执行(详情参见页面上的链路图，一个callback跟一个errback对应，每个callback(errback)的结果传递给下一个callback/errback，同级的errback(callback)没机会执行了)，使用errback返回的结果。如果确实return a twisted.python.failure.Failure或者raise an exception，那么错误会传递给下一个errback.

如果一个errback不返回任何东西，就等效于返回了一个None对象，那么意味着callbacks会在这errback之后继续执行。callback会用None作为参数，所以要小心！

twisted.python.failure.Failure实例有一个有用的函数叫trap,可用于作如下等效行为：

try:
    # code that may throw an exception
    cookSpamAndEggs()
except (SpamException, EggException):
    # Handle SpamExceptions and EggExceptions
    ...
等效于

def errorHandler(failure):
    failure.trap(SpamException, EggException)
    # Handle SpamExceptions and EggExceptions
d.addCallback(cookSpamAndEggs)
d.addErrback(errorHandler)
如果传递给trap的参数跟Failure中封装的错误不匹配，trap会re-raise那个错误。

twisted.internet.defer.Deferred.addCallbacks()

# Case 1
d = getDeferredFromSomewhere()
d.addCallback(callback1)       # A
d.addErrback(errback1)         # B
d.addCallback(callback2)       
d.addErrback(errback2)        
# Case 2
d = getDeferredFromSomewhere()
d.addCallbacks(callback1, errback1)  # C
d.addCallbacks(callback2, errback2)
如果callback1中发生错误，Case 1中errback1会被调用来处理Failure；Case 2中errback2会被调用来处理Failure。

在Case 1中，line A的callback会处理getDeferredFromSomewhere中的成功结果，line B的errback会处理上流代码(the upstream source)中的错误或者A中的错误。

在Case 2中，line C的errback只处理getDeferredFromSomewhere中抛出的错误，不会处理callback1中抛出的错误。

对于没有添加errback而发生错误的情况，Deferred被垃圾回收的时候，twisted会把错误的调用栈(traceback)打印到日志文件。所以要小心，如果我们保留了一个对Deferred的引用，从而阻止了它被垃圾回收，那么我们永远都看不到这个错误 （并且会发现我们的callback很神秘的从不被调用）。如果不确定是否有errback忘写，可以显式的在我们callbacks后面追加一个如下所示的errback：

from twisted.python import log
d.addErrback(log.err)
Handling either synchronous or asynchronous results 
有些应用程序中，函数要嘛是异步的(返回一个Deferred,当数据到达时fire这个Deferred)要嘛是同步的(立马返回一个结果)。不过，也有些函数需要同时接受立即结果和Deferreds.

同步函数：

def authenticateUser(isValidUser, user):
    if isValidUser(user):
        print "User is authenticated"
    else:
        print "User is not authenticated"
异步函数：
from twisted.internet import reactor, defer
def asynchronousIsValidUser(user):
    d = defer.Deferred()
    reactor.callLater(2, d.callback, user in ["Alice", "Angus", "Agnes"])
    return d
twisted.internet.defer.maybeDeferred()可以保证函数的返回值是一个Deferred，即使函数是一个同步函数：
from twisted.internet import defer
def printResult(result):
  if result:
    print "User is authenticated"
  else:
    print "User is not authenticated"
def authenticateUser(isValidUser, user):
  d = defer.maybeDeferred(isValidUser, user)
  d.addCallback(printResult)
isValidUser是synchronousIsValidUser或者asynchronousIsValidUser都行。

当然，我们也可以修改synchronousIsValidUser使其返回一个Deferred.(https://twistedmatrix.com/documents/current/core/howto/gendefer.html)

有时候我们需要在几个不同的事件都发生了才被通知，而不是分别等待每个事件。例如，我们可能需要等待一个列表中的连接全部关闭。 twisted.internet.defer.DeferredList就是干这个的。

根据多个Deferred来创建一个DeferredList，我们只需简单的把一个存储着目标Deferred的list传递进去就行了：

dl = defer.DeferredList([deferred1,deferred2,deferred3])
现在我们可以把DeferredList当成一个普通的Deferred对待；我们可以调用addCallbacks等方法。 DeferredList会在所有Deferreds完成的时候调用它的callback. callback会以DeferredList包含的所有Deferred的结果构成的list作为参数进行执行 ，like so:

# A callback that unpacks and prints the results of a DeferredList
def printResult(result):
  for (success, value) in result:
    if success:
      print 'Success:', value
    else:
      print 'Failure:', value.getErrorMessage()

# Create three deferreds.
deferred1 = defer.Deferred()
deferred2 = defer.Deferred()
deferred3 = defer.Deferred()

# Pack them into a DeferredList
dl = defer.DeferredList([deferred1, deferred2, deferred3], consumeErrors=True)

# Add our callback
dl.addCallback(printResult)

# Fire our three deferreds with various values.
deferred1.callback('one')
deferred2.errback(Exception('bang!'))
deferred3.callback('three')

# At this point, dl will fire its callback, printing:
#	Success: one
#	Failure: bang!
#	Success: three
# (note that defer.SUCCESS == True, and defer.FAILURE == False)
标准的DeferredList不会调用errback,但是传给DeferredList的Deferred发生的failure还是会调用errback除非DeferredList的consumeErrors被设成了True(这句话很绕，我理解的意思是，DeferredList只会等待所有Deferred的异步数据到大后才会执行callback,但是不会执行errback；而构成DeferredList的Deferred发生错误时还是可以调用自己的errback；要想阻止Deferred调用自己的errback,需要在构建DeferredList时设置参数consumeError=True.这个理解需要回头用程序验证一下！！)。
如果要为构成DeferredList的独立的Deffered提供callback，我们应该对于这些callback添加的时间点很小心。添加一个Deferred到DeferedList的行为实际上内部为Deferred插入了一个callback(当这个callback执行的时候，它检查DeferredList是否已经完成了(我理解的意思是，这个callback检测自己是否是最后一个完成的Deferred，如果是的话就可以调用DefferredList的callback了)).要记住的很重要的事情是，这个内部自动加入的callback它记录的值会是作为result list传入给DefferedList callback函数的东东。

因此，如果我们在把Deferred加入到DeferredList之后再添加一个callback给Deferred，那么这个callback返回的值将不会传给DeferredList的callback（擦，这么说的话，DeferredList不但要等待它内部所有Deferred的数据到达，而且还要等所有Deferred的callback执行完成，然后把最后的结果合一起作为DeferredList等待的数据）. 为了避免困惑，我们建议不要在Deferred添加到DeferredList之后再为Deferred添加callback.

def printResult(result):
  print result
def addTen(result):
  return result + " ten"

# Deferred gets callback before DeferredList is created
deferred1 = defer.Deferred()
deferred2 = defer.Deferred()
deferred1.addCallback(addTen)
dl = defer.DeferredList([deferred1, deferred2])
dl.addCallback(printResult)
deferred1.callback("one") # fires addTen, checks DeferredList, stores "one ten"
deferred2.callback("two")
# At this point, dl will fire its callback, printing:
#	 [(1, 'one ten'), (1, 'two')]

# Deferred gets callback after DeferredList is created
deferred1 = defer.Deferred()
deferred2 = defer.Deferred()
dl = defer.DeferredList([deferred1, deferred2])
deferred1.addCallback(addTen) # will run *after* DeferredList gets its value
dl.addCallback(printResult)
deferred1.callback("one") # checks DeferredList, stores "one", fires addTen
deferred2.callback("two")
# At this point, dl will fire its callback, printing:
#	 [(1, 'one), (1, 'two')]
Other behaviours 
fireOnOneCallback DeferredList会在有Deferred调用callback时立马调用自己的callback,然后DeferredList不做任何事儿了，忽略其他还没完成的Deferred

fireOnOneErrback DeferredList会在有Deferred调用errback时立马调用自己的errback,然后DeferredList不做任何事儿了，忽略其他还没完成的Deferred

consumeErrors 阻止错误在DeferredList包含的任何Deferred的链路上传播（普通创建的DeferredList不会影响结果在它的Deferred的callbacks和errbacks中传递）。用这个选项在DeferedList中停止错误，将阻止“Unhandled error in Deferred”警告出现在DeferredList包含的没必要添加额外errbacks的Deferred中。设置consumeErrors参数为true不会改变fireOnOneCallback或者fireOnOneErrback.

DeferredList的一个常见应用是join若干平行的异步操作，如果所有操作都成功就成功完成，如果任何一个操作失败就失败。这种情况下，twisted.internet.defer.gatherResults是一个有用的捷径：

from twisted.internet import defer
d1 = defer.Deferred()
d2 = defer.Deferred()
d = defer.gatherResults([d1, d2], consumeErrors=True)
def printResult(result):
  print result
d.addCallback(printResult)
d1.callback("one")
# nothing is printed yet; d is still awaiting completion of d2
d2.callback("two")
# printResult prints ["one", "two"]
consumeErrors参数跟DeferredList有一样的意义：如果为true,gatherResults会消耗掉传进来的Deferred的所有错误。这个参数都设为true吧，除非我们要为传进来的Deferred添加callbacks或者errbacks,或者除非我们知道他们不会失败。如果不设为true,失败会导致twisted记录一个unhandled error到日志里。这个参数从twisted 11.1.0之后有效。