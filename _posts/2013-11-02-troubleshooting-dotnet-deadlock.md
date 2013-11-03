---
layout: post
title: 调试.Net程序中出现的死锁
tags : [DotNet, Deadlock, Debugging]
---
{% include JB/setup %}

前段时间在生产环境中遇到一个.Net程序[死锁](http://zh.wikipedia.org/wiki/%E6%AD%BB%E9%94%81)的问题，IIS的应用程序池在回收以后偶尔会出现不响应所有请求的情况。通过调试最后发现一个死锁问题造成的。


## 调试过程

死锁是由两个以上的线程互相等待对方释放资源引起的，死锁相对其它Bug而言比较隐晦，调试也比较困难。死锁一般在并发环境中才会出现，在开发环境里很难复现，在生产环境中又无法向进行单步调试。因此，抓取Dump文件，然后用Windug分析是一个不错的方法。在Windug中调试.Net程序需要[SOS调试扩展(SOS.dll)](http://msdn.microsoft.com/zh-cn/library/bb190764(v=vs.110\).aspx)。SOS.dll在.NET Framework的安装目录中可以找到，不同的.Net版本对应不同的SOS.dll。

首先确定程序使用的.Net版本，加载SOS.dll。

    0:000> lm vm mscorwks
    start             end                 module name
    000007fe`f1d30000 000007fe`f26cc000   mscorwks   (deferred)             
        Image path: C:\Windows\Microsoft.NET\Framework64\v2.0.50727\mscorwks.dll
        Image name: mscorwks.dll
        Timestamp:        Thu Aug 30 13:18:34 2012 (503EF7AA)
        CheckSum:         00987FE1
        ImageSize:        0099C000
        File version:     2.0.50727.5466
        Product version:  2.0.50727.5466
        File flags:       0 (Mask 3F)
        File OS:          4 Unknown Win32
        File type:        2.0 Dll
        File date:        00000000.00000000
        Translations:     0409.04b0
        CompanyName:      Microsoft Corporation
        ProductName:      Microsoft® .NET Framework
        InternalName:     mscorwks.dll
        OriginalFilename: mscorwks.dll
        ProductVersion:   2.0.50727.5466
        FileVersion:      2.0.50727.5466 (Win7SP1GDR.050727-5400)
        FileDescription:  Microsoft .NET Runtime Common Language Runtime - WorkStation
        LegalCopyright:   © Microsoft Corporation.  All rights reserved.
        Comments:         Flavor=Retail

    0:000> .load C:\Windows\Microsoft.NET\Framework64\v2.0.50727\SOS.dll

然后可以通过`syncblk`命令查看sync block来发现那些线程拥有锁，哪些线程等待锁。通过syncblk可以看到目前有一个sync block已经被线程49（系统线程28a4）所拥有。

    0:000> !syncblk
    Index         SyncBlock MonitorHeld Recursion Owning Thread Info                  SyncBlock Owner
       60 0000000001d9a330          309         2 00000000072437d0  28a4  49   000000011ff08068 System.Object
    -----------------------------
    Total           175
    CCW             3
    RCW             3
    ComClassFactory 0
    Free            0

切换到49号线程，验证系统线程号是否为28a4。

    0:000> ~49s
    ntdll!ZwWaitForSingleObject+0xa:
    00000000`77a7135a c3              ret
    0:049> ~.
    . 49  Id: 20ec.28a4 Suspend: 0 Teb: 000007ff`ffe9e000 Unfrozen
          Start: mscorwks!Thread::intermediateThreadProc     (000007fe`f1e1b564) 
          Priority: 0  Priority class: 32  Affinity: ffff

输出其调用栈，原来该线程在等待进入另外一个CriticalSection。

    0:049> kvL
    Child-SP          RetAddr           : Args to Child                                                           : Call Site
    00000000`0869c3e8 00000000`77a6e4e8 : 00000000`00000000 00000000`00000ab8 0000232c`7452da4b 00000000`0869ca58 : ntdll!ZwWaitForSingleObject+0xa
    00000000`0869c3f0 00000000`77a6e3db : 00000000`00000000 00000000`072437d0 00000000`00000001 00000000`07339ba0 : ntdll!RtlpWaitOnCriticalSection+0xe8
    00000000`0869c4a0 000007fe`f1fe07d4 : 00000000`0869d2d0 00000000`0000000a 00000000`07339ba0 00000000`0869d2d0 : ntdll!RtlEnterCriticalSection+0xd1
    00000000`0869c4d0 000007fe`f23f1373 : 00000000`01dded20 00000000`073b2580 ffffffff`fffffffe 000007ff`002d1ef0 : mscorwks!UnsafeEEEnterCriticalSection+0x20
    00000000`0869c500 000007fe`f1f650a3 : 00000000`00000000 000007fe`f2483a00 ffffffff`fffffffe 00000000`00000000 : mscorwks!CrstBase::Enter+0x123
    00000000`0869c580 000007fe`f1f3d36a : ffffffff`00000001 00000000`07203140 00000000`00000000 00000000`00000001 : mscorwks!ListLockEntry::FinishDeadlockAwareEnter+0x2b
    00000000`0869c5c0 000007fe`f1e8837b : 00000000`0869c6f0 00000000`0720e110 00000000`00000000 00000000`00000000 : mscorwks!ListLockEntry::LockHolder::DeadlockAwareAcquire+0x32
    00000000`0869c5f0 000007fe`f2374198 : 00000000`00000000 000007ff`002c77a8 00000000`00000008 000007ff`001bc5e0 : mscorwks!MethodTable::DoRunClassInitThrowing+0x6cb
    00000000`0869d070 000007fe`f23a1dca : ffffffff`00000001 00000000`02c1ac60 00000000`00000000 00000000`072437d0 : mscorwks!MethodTable::CheckRunClassInitThrowing+0x68
    00000000`0869d0b0 000007ff`001e86c8 : 000007ff`001bc5e0 00000000`00000008 00000000`00000000 00000000`00000001 : mscorwks!JIT_GetSharedNonGCStaticBase_Helper+0xfa
    00000000`0869d270 000007fe`f1feeb52 : 00000001`dff82aa0 00000000`0869ca18 ffffffff`fffffffe 00000000`072437d0 : 0x7ff`001e86c8
    00000000`0869d2a0 000007fe`f1e5dea3 : 00000000`0720e050 00000000`0869d760 00000000`00000000 00000000`00000000 : mscorwks!CallDescrWorker+0x82
    00000000`0869d2f0 000007fe`f23ca841 : 00000000`0869d428 00000000`00000000 00000000`00000000 00000000`00000001 : mscorwks!CallDescrWorkerWithHandler+0xd3
    00000000`0869d390 000007fe`f1f2b371 : 00000000`0869d7f0 000007fe`f1e791ee 00000000`00000000 000007fe`f1e8ae25 : mscorwks!MethodDesc::CallDescr+0x2b1
    00000000`0869d5d0 000007fe`f1ef4913 : 00000000`00000000 00000000`0869d670 00000000`0869d650 000007ff`002c7a20 : mscorwks!MethodDescCallSite::CallWithValueTypes+0x35
    00000000`0869d610 000007fe`f244d8b2 : 00000000`072437d0 000007fe`f240ead0 000007ff`002c7a20 000007ff`002c7a20 : mscorwks!InvokeConstructorHelper+0x36b
    SYMSRV:  mscorlib.pdb from http://msdl.microsoft.com/download/symbols: 117553 bytes - copied         
    *** WARNING: Unable to verify checksum for mscorlib.ni.dll
    DBGHELP: mscorlib_ni - public symbols  
             d:\windbgsymbols\mscorlib.pdb\4FAE2D4BA0114F37ACE06598025C0D491\mscorlib.pdb
    00000000`0869d950 000007fe`f0bc4a82 : 00000000`00000000 000007ff`002c7a10 00000000`0869dc40 00000001`00000000 : mscorwks!RuntimeMethodHandle::InvokeConstructor+0x112
    00000000`0869db30 000007ff`001e7d1d : 00000001`dff82508 00000001`dff824e0 00000000`00000000 000007ff`00350cb4 : mscorlib_ni+0x374a82
    00000000`0869dcd0 000007ff`001e6293 : 00000001`dff823c8 000007ff`001dbbc7 4296801a`6b2f6a09 86753423`04f42faf : 0x7ff`001e7d1d
    00000000`0869dd90 000007ff`001dbaa0 : 0000000d`fffffffe 00000001`dff7f5e0 4296801a`6b2f6a09 86753423`04f42faf : 0x7ff`001e6293

将该CriticalSection输出，查看一下拥有的线程是26a4，即51号线程。

    0:049> !cs 07339ba0
    -----------------------------------------
    Critical section   = 0x0000000007339ba0 (+0x7339BA0)
    DebugInfo          = 0x0000000007356940
    LOCKED
    LockCount          = 0x5
    WaiterWoken        = No
    OwningThread       = 0x00000000000026a4
    RecursionCount     = 0x1
    LockSemaphore      = 0xAB8
    SpinCount          = 0x0000000000000000

    0:049> ~
    . 49  Id: 20ec.28a4 Suspend: 0 Teb: 000007ff`ffe9e000 Unfrozen
      50  Id: 20ec.2188 Suspend: 0 Teb: 000007ff`ffe9c000 Unfrozen
      51  Id: 20ec.26a4 Suspend: 0 Teb: 000007ff`ffe9a000 Unfrozen

切换到51号线程查看调用栈，可以看到该调用栈正在等待mscorwks!AwareLock::Enter，即49号线程拥有的sync block。

    0:049> ~51s
    ntdll!ZwWaitForMultipleObjects+0xa:
    00000000`77a718ca c3              ret

    0:051> kL
    Child-SP          RetAddr           Call Site
    00000000`08d4b268 000007fe`fdac1430 ntdll!ZwWaitForMultipleObjects+0xa
    00000000`08d4b270 00000000`77922ce3 KERNELBASE!WaitForMultipleObjectsEx+0xe8
    00000000`08d4b370 000007fe`f1f3efc5 kernel32!WaitForMultipleObjectsExImplementation+0xb3
    00000000`08d4b400 000007fe`f1f3a825 mscorwks!WaitForMultipleObjectsEx_SO_TOLERANT+0xc1
    00000000`08d4b4a0 000007fe`f1e3a75d mscorwks!Thread::DoAppropriateAptStateWait+0x41
    00000000`08d4b500 000007fe`f1f1ebf4 mscorwks!Thread::DoAppropriateWaitWorker+0x191
    00000000`08d4b600 000007fe`f1eddfde mscorwks!Thread::DoAppropriateWait+0x5c
    00000000`08d4b670 000007fe`f1f5bf69 mscorwks!CLREvent::WaitEx+0xbe
    00000000`08d4b720 000007fe`f1e0cc7a mscorwks!AwareLock::EnterEpilog+0xc9
    00000000`08d4b7f0 000007fe`f2388a25 mscorwks!AwareLock::Enter+0x72
    00000000`08d4b820 000007ff`001da8a5 mscorwks!JIT_MonEnterWorker_Portable+0xf5
    00000000`08d4b9f0 000007ff`001da81a 0x7ff`001da8a5
    00000000`08d4ba70 000007ff`001d886c 0x7ff`001da81a
    00000000`08d4baa0 000007ff`001d86cb 0x7ff`001d886c
    00000000`08d4bb00 000007ff`001d85b3 0x7ff`001d86cb
    00000000`08d4bb50 000007ff`001d854a 0x7ff`001d85b3
    00000000`08d4bb90 000007ff`001e72b9 0x7ff`001d854a
    00000000`08d4bbe0 000007fe`f1feeb52 0x7ff`001e72b9
    00000000`08d4bc20 000007fe`f1e5dea3 mscorwks!CallDescrWorker+0x82
    00000000`08d4bc60 000007fe`f1e5dd56 mscorwks!CallDescrWorkerWithHandler+0xd3

因此我们可以看到是49号线程和51号线程互相得到了彼此需要请求的锁，因而造成了死锁。

让我们分别看一下49号线程和51号线程的托管线程栈。

    0:049> ~49e!clrstack
    OS Thread Id: 0x28a4 (49)
    Child-SP         RetAddr          Call Site
    000000000869d270 000007fef1feeb52 BitAuto.AutoAlbum.Logic.DataStorage.DataStorageService..ctor()
    000000000869db30 000007ff001e7d1d System.Reflection.RuntimeConstructorInfo.Invoke(System.Reflection.BindingFlags, System.Reflection.Binder, System.Object[], System.Globalization.CultureInfo)
    000000000869dcd0 000007ff001e6293 Autofac.Core.Activators.Reflection.ConstructorParameterBinding.Instantiate()
    000000000869dd90 000007ff001dbaa0 Autofac.Core.Activators.Reflection.ReflectionActivator.ActivateInstance(Autofac.IComponentContext, System.Collections.Generic.IEnumerable`1<Autofac.Core.Parameter>)
    000000000869ddf0 000007ff001dcc64 Autofac.Core.Resolving.InstanceLookup.Activate(System.Collections.Generic.IEnumerable`1<Autofac.Core.Parameter>)
    000000000869de30 000007ff001db928 Autofac.Core.Lifetime.LifetimeScope.GetOrCreateAndShare(System.Guid, System.Func`1<System.Object>)
    000000000869ded0 000007ff001daf08 Autofac.Core.Resolving.InstanceLookup.Execute()
    000000000869df40 000007ff001e7fbb Autofac.Core.Resolving.ResolveOperation.GetOrCreateInstance(Autofac.Core.ISharingLifetimeScope, Autofac.Core.IComponentRegistration, System.Collections.Generic.IEnumerable`1<Autofac.Core.Parameter>)
    000000000869dfa0 000007ff001e7f59 Autofac.Core.Resolving.InstanceLookup.ResolveComponent(Autofac.Core.IComponentRegistration, System.Collections.Generic.IEnumerable`1<Autofac.Core.Parameter>)
    000000000869dfd0 000007ff001e7cd7 Autofac.Core.Activators.Reflection.AutowiringParameter+<>c__DisplayClass2.<CanSupplyValue>b__0()
    000000000869e010 000007ff001e6293 Autofac.Core.Activators.Reflection.ConstructorParameterBinding.Instantiate()
    000000000869e0d0 000007ff001dbaa0 Autofac.Core.Activators.Reflection.ReflectionActivator.ActivateInstance(Autofac.IComponentContext, System.Collections.Generic.IEnumerable`1<Autofac.Core.Parameter>)
    000000000869e130 000007ff001dcc64 Autofac.Core.Resolving.InstanceLookup.Activate(System.Collections.Generic.IEnumerable`1<Autofac.Core.Parameter>)
    000000000869e170 000007ff001db928 Autofac.Core.Lifetime.LifetimeScope.GetOrCreateAndShare(System.Guid, System.Func`1<System.Object>)
    000000000869e210 000007ff001daf08 Autofac.Core.Resolving.InstanceLookup.Execute()
    000000000869e280 000007ff001dacfc Autofac.Core.Resolving.ResolveOperation.GetOrCreateInstance(Autofac.Core.ISharingLifetimeScope, Autofac.Core.IComponentRegistration, System.Collections.Generic.IEnumerable`1<Autofac.Core.Parameter>)
    000000000869e2e0 000007ff001da903 Autofac.Core.Resolving.ResolveOperation.Execute(Autofac.Core.IComponentRegistration, System.Collections.Generic.IEnumerable`1<Autofac.Core.Parameter>)
    000000000869e350 000007ff001d886c Autofac.Core.Lifetime.LifetimeScope.ResolveComponent(Autofac.Core.IComponentRegistration, System.Collections.Generic.IEnumerable`1<Autofac.Core.Parameter>)
    000000000869e3d0 000007ff001d86cb Autofac.ResolutionExtensions.TryResolveService(Autofac.IComponentContext, Autofac.Core.Service, System.Collections.Generic.IEnumerable`1<Autofac.Core.Parameter>, System.Object ByRef)
    000000000869e430 000007ff001dee38 Autofac.ResolutionExtensions.ResolveService(Autofac.IComponentContext, Autofac.Core.Service, System.Collections.Generic.IEnumerable`1<Autofac.Core.Parameter>)
    000000000869e480 000007ff001dec48 Autofac.Core.Activators.Reflection.AutowiringPropertyInjector.InjectProperties(Autofac.IComponentContext, System.Object, Boolean)
    000000000869e500 000007ff001dead6 Autofac.ResolutionExtensions.InjectProperties[[System.__Canon, mscorlib]](Autofac.IComponentContext, System.__Canon)
    000000000869e540 000007ff001de273 Autofac.Integration.Web.Forms.PageInjectionBehaviour.DoInjection(System.Func`2<System.Object,System.Object>, System.Object)
    000000000869e590 000007feed6a59b0 Autofac.Integration.Web.Forms.DependencyInjectionModule.OnPreRequestHandlerExecute(System.Object, System.EventArgs)
    000000000869e5d0 000007feed69671b System.Web.HttpApplication+SyncEventExecutionStep.System.Web.HttpApplication.IExecutionStep.Execute()
    000000000869e630 000007feedd83971 System.Web.HttpApplication.ExecuteStep(IExecutionStep, Boolean ByRef)
    000000000869e6d0 000007feedd743b2 System.Web.HttpApplication+PipelineStepManager.ResumeSteps(System.Exception)
    000000000869e860 000007feedd55df9 System.Web.HttpApplication.BeginProcessRequestNotification(System.Web.HttpContext, System.AsyncCallback)
    000000000869e8b0 000007feede7dde1 System.Web.HttpRuntime.ProcessRequestNotificationPrivate(System.Web.Hosting.IIS7WorkerRequest, System.Web.HttpContext)
    000000000869e9d0 000007feede7d9ab System.Web.Hosting.PipelineRuntime.ProcessRequestNotificationHelper(IntPtr, IntPtr, IntPtr, Int32)
    000000000869eb50 000007feede7cc14 System.Web.Hosting.PipelineRuntime.ProcessRequestNotification(IntPtr, IntPtr, IntPtr, Int32)
    000000000869ebb0 000007fef1fec6ca DomainNeutralILStubClass.IL_STUB(Int64, Int64, Int64, Int32)
    000000000869f3e0 000007feede7df10 DomainNeutralILStubClass.IL_STUB(IntPtr, System.Web.RequestNotificationStatus ByRef)
    000000000869f4c0 000007feede7d9ab System.Web.Hosting.PipelineRuntime.ProcessRequestNotificationHelper(IntPtr, IntPtr, IntPtr, Int32)
    000000000869f640 000007feede7cc14 System.Web.Hosting.PipelineRuntime.ProcessRequestNotification(IntPtr, IntPtr, IntPtr, Int32)
    000000000869f6a0 000007fef1fec91b DomainNeutralILStubClass.IL_STUB(Int64, Int64, Int64, Int32)

    0:049> ~51e!clrstack
    OS Thread Id: 0x26a4 (51)
    Child-SP         RetAddr          Call Site
    0000000008d4b9f0 000007ff001da81a Autofac.Core.Lifetime.LifetimeScope.ResolveComponent(Autofac.Core.IComponentRegistration, System.Collections.Generic.IEnumerable`1<Autofac.Core.Parameter>)
    0000000008d4ba70 000007ff001d886c Autofac.Core.Container.ResolveComponent(Autofac.Core.IComponentRegistration, System.Collections.Generic.IEnumerable`1<Autofac.Core.Parameter>)
    0000000008d4baa0 000007ff001d86cb Autofac.ResolutionExtensions.TryResolveService(Autofac.IComponentContext, Autofac.Core.Service, System.Collections.Generic.IEnumerable`1<Autofac.Core.Parameter>, System.Object ByRef)
    0000000008d4bb00 000007ff001d85b3 Autofac.ResolutionExtensions.ResolveService(Autofac.IComponentContext, Autofac.Core.Service, System.Collections.Generic.IEnumerable`1<Autofac.Core.Parameter>)
    0000000008d4bb50 000007ff001d854a Autofac.ResolutionExtensions.Resolve[[System.__Canon, mscorlib]](Autofac.IComponentContext, System.Collections.Generic.IEnumerable`1<Autofac.Core.Parameter>)
    0000000008d4bb90 000007ff001e72b9 Autofac.ResolutionExtensions.Resolve[[System.__Canon, mscorlib]](Autofac.IComponentContext)
    0000000008d4bbe0 000007fef1feeb52 BitAuto.AutoAlbum.Logic.BaseService..cctor()
    0000000008d4cb30 000007ff001e6c3f BitAuto.AutoAlbum.Logic.Car.CarService..ctor()
    0000000008d4cb80 000007fef1feeb52 BitAuto.AutoAlbum.GalleryWeb.API.GetColorsInCarYear..cctor()
    0000000008d4ddc0 000007fef0b6242a System.RuntimeType.CreateInstanceSlow(Boolean, Boolean)
    0000000008d4de40 000007fef0b4182f System.RuntimeType.CreateInstanceImpl(Boolean, Boolean, Boolean)
    0000000008d4ded0 000007feee15c65f System.Activator.CreateInstance(System.Type, Boolean)
    0000000008d4df10 000007feedef028b System.Web.Compilation.BuildResultCompiledType.CreateInstance()
    0000000008d4df40 000007feedef0203 System.Web.UI.SimpleHandlerFactory.System.Web.IHttpHandlerFactory2.GetHandler(System.Web.HttpContext, System.String, System.Web.VirtualPath, System.String)
    0000000008d4df90 000007feedd77d59 System.Web.UI.SimpleHandlerFactory.GetHandler(System.Web.HttpContext, System.String, System.String, System.String)
    0000000008d4dfe0 000007feed696777 System.Web.HttpApplication+MaterializeHandlerExecutionStep.System.Web.HttpApplication.IExecutionStep.Execute()
    0000000008d4e0a0 000007feedd83971 System.Web.HttpApplication.ExecuteStep(IExecutionStep, Boolean ByRef)
    0000000008d4e140 000007feedd743b2 System.Web.HttpApplication+PipelineStepManager.ResumeSteps(System.Exception)
    0000000008d4e2d0 000007feedd55df9 System.Web.HttpApplication.BeginProcessRequestNotification(System.Web.HttpContext, System.AsyncCallback)
    0000000008d4e320 000007feede7dde1 System.Web.HttpRuntime.ProcessRequestNotificationPrivate(System.Web.Hosting.IIS7WorkerRequest, System.Web.HttpContext)
    0000000008d4e440 000007feede7d9ab System.Web.Hosting.PipelineRuntime.ProcessRequestNotificationHelper(IntPtr, IntPtr, IntPtr, Int32)
    0000000008d4e5c0 000007feede7cc14 System.Web.Hosting.PipelineRuntime.ProcessRequestNotification(IntPtr, IntPtr, IntPtr, Int32)
    0000000008d4e620 000007fef1fec6ca DomainNeutralILStubClass.IL_STUB(Int64, Int64, Int64, Int32)
    0000000008d4ee50 000007feede7df10 DomainNeutralILStubClass.IL_STUB(IntPtr, System.Web.RequestNotificationStatus ByRef)
    0000000008d4ef30 000007feede7d9ab System.Web.Hosting.PipelineRuntime.ProcessRequestNotificationHelper(IntPtr, IntPtr, IntPtr, Int32)
    0000000008d4f0b0 000007feede7cc14 System.Web.Hosting.PipelineRuntime.ProcessRequestNotification(IntPtr, IntPtr, IntPtr, Int32)
    0000000008d4f110 000007fef1fec91b DomainNeutralILStubClass.IL_STUB(Int64, Int64, Int64, Int32)

从这里可以看到49号线程调用`BitAuto.AutoAlbum.Logic.DataStorage.DataStorageService`的构造函数和51号线程调用`Autofac.Core.Lifetime.LifetimeScope.ResolveComponent`方法，互相等待对方释放锁。

定位到具体的代码就好办了，查看程序代码，看看是怎么造成的。首先看一下49号线程调用的代码。

	class DataStorageService : BaseService, IDataStorageService
	{
		public DataStorageService()
		{
			int memoryCacheTimeInSecond;
			int.TryParse(ConfigurationManager.AppSettings["MemoryCacheTime"], out memoryCacheTimeInSecond);
			int memcachedCacheTimeInSecond;
			int.TryParse(ConfigurationManager.AppSettings["MemcachedCacheTime"], out memcachedCacheTimeInSecond);
			_cacheStrategy = new MemoryAndMemcachedCacheStrategy(memoryCacheTimeInSecond, memcachedCacheTimeInSecond);
		}
	}

可以看到DataStorageService的构造函数并没有能够引起死锁的代码，再看一下51号线程调用的代码，51号线程调用了BaseService的静态构造函数。

	public class BaseService
	{
		static BaseService()
		{
			int expirationSecond;
			int.TryParse(ConfigurationManager.AppSettings["DataCacheTimeFromSecond"], out expirationSecond);
			cacheContainer = new WebCacheContainer(expirationSecond);
			m_logger = DependencyManager.Instance.Container.Resolve<ILog>();
		}
	}

在这里发现`DependencyManager.Instance.Container.Resolve<ILog>()`程调用的是Ioc容器[Autofac](https://code.google.com/p/autofac/)的代码，问题很有可能就出在这儿。
再看一下`Resolve`方法的实现，`Resolve`在内部调用了`Autofac.Core.Lifetime.LifetimeScope.ResolveComponent`方法，它的代码是这样的，我们看到了一个熟悉的lock。

        /// <summary>
        /// Resolve an instance of the provided registration within the context.
        /// </summary>
        /// <param name="registration">The registration.</param>
        /// <param name="parameters">Parameters for the instance.</param>
        /// <returns>
        /// The component instance.
        /// </returns>
        /// <exception cref="Autofac.Core.Registration.ComponentNotRegisteredException"/>
        /// <exception cref="DependencyResolutionException"/>
        public object ResolveComponent(IComponentRegistration registration, IEnumerable<Parameter> parameters)
        {
            if (registration == null) throw new ArgumentNullException("registration");
            if (parameters == null) throw new ArgumentNullException("parameters");
            CheckNotDisposed();

            lock (_synchRoot)
            {
                var operation = new ResolveOperation(this);
                ResolveOperationBeginning(this, new ResolveOperationBeginningEventArgs(operation));
                return operation.Execute(registration, parameters);
            }
        } 

回头看一下49号的线程栈，发现DataStorageService的构造函数是被`Autofac.Core.Lifetime.LifetimeScope.GetOrCreateAndShare`方法调用的，它的代码是这样的。

        /// <summary>
        /// Try to retrieve an instance based on a GUID key. If the instance
        /// does not exist, invoke <paramref name="creator"/> to create it.
        /// </summary>
        /// <param name="id">Key to look up.</param>
        /// <param name="creator">Creation function.</param>
        /// <returns>An instance.</returns>
        public object GetOrCreateAndShare(Guid id, Func<object> creator)
        {
            lock (_synchRoot)
            {
                object result;
                if (!_sharedInstances.TryGetValue(id, out result))
                {
                    result = creator();
                    _sharedInstances.Add(id, result);
                }
                return result;
            }
        }

可以看到它和上面的`ResolveComponent`使用的同一个线程锁，终于可以确定问题了。49号线程获得了`LifetimeScope`的线程锁，创建`DataStorageService`的实例时等待父类`BaseService`静态构造函数的sync block。51号线程获得了`BaseService`静态构造函数的sync block，等待`LifetimeScope`的线程锁。两个线程互相等待对方释放资源，造成了死锁。由于`BaseService`是所有Service类的基类，只要51号线程不释放sync block，所有的Service类都无法创建，从而阻塞了所有线程的执行。

## 解决问题

找到问题解决就很简单了，把`Resolve`方法的调用从`BaseService`静态构造函数中挪出来放到实例构造函数中就可以了。

##总结

在静态构造函数中调用代码一定要谨慎，特别是像Ioc容器这样创建实例的类库，避免可能互相等待的资源竞争，死锁就不会发生了。