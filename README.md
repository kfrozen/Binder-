## Binder和跨进程通信的那些事 ##

Binder是Android系统中实现的一种高效的IPC机制，用于实现跨进程通信。

- **Binder机制完成跨进程通信的大致流程**：

	1. 首先，Server进程要向ServiceManager(SM)注册；告诉自己是谁，自己有什么能力；在这个场景就是Server告诉SM，它叫zhangsan，它有一个object对象，可以执行add 操作；于是SM建立了一张表：zhangsan这个名字对应进程Server;
	2. 然后Client向SM查询：我需要联系一个名字叫做zhangsan的进程里面的object对象；这时候关键来了：进程之间通信的数据都会经过运行在内核空间里面的驱动，驱动在数据流过的时候做了一点手脚，它并不会给Client进程返回一个真正的object对象，而是返回一个看起来跟object一模一样的代理对象objectProxy，这个objectProxy也有一个add方法，但是这个add方法没有Server进程里面object对象的add方法那个能力；objectProxy的add只是一个傀儡，它唯一做的事情就是把参数包装然后交给驱动。
	3. 但是Client进程并不知道驱动返回给它的对象动过手脚，毕竟伪装的太像了，如假包换。Client开开心心地拿着objectProxy对象然后调用add方法；我们说过，这个add什么也不做，直接把参数做一些包装然后直接转发给Binder驱动。
	4. 驱动收到这个消息，发现是这个objectProxy；一查表就明白了：我之前用objectProxy替换了object发送给Client了，它真正应该要访问的是object对象的add方法；于是Binder驱动通知Server进程，调用你的object对象的add方法，然后把结果发给我，Sever进程收到这个消息，照做之后将结果返回驱动，驱动然后把结果返回给Client进程；于是整个过程就完成了。
	
- **Java层的Binder**

	这里有几个基本的接口：

	- **IBinder**是一个接口，它代表了一种**跨进程传输的能力**；只要实现了这个接口，就能将这个对象进行跨进程传递；这是驱动底层支持的；在跨进程数据流经驱动的时候，驱动会识别IBinder类型的数据，从而自动完成不同进程Binder本地对象以及Binder代理对象的转换。
	- IBinder负责数据传输，那么client与server端的调用契约（这里不用接口避免混淆）呢？这里的**IInterface**代表的就是远程server对象具有什么能力。具体来说，就是aidl里面的接口。
	- Java层的Binder类，代表的其实就是**Binder本地对象**。**BinderProxy**类是Binder类的一个内部类，它代表远程进程的Binder对象的本地代理；这两个类都继承自IBinder, 因而都具有跨进程传输的能力；实际上，在跨越进程的时候，Binder驱动会自动完成这两个对象的转换。

- **Binder本地对象和Binder代理对象**：

	跨进程通信中我们可以把两端的进程看做一个C/S模型，发起请求的是Client，处理请求的是Server。与此对应的，Server端的Binder指的是**Binder本地对象**，而Client端的Binder指的是**Binder代理对象**，即Server端Binder对象的一个远程代理。我们通过ActivityManager相关的源码来看看这两个东西是怎么来的：

	1. 先看一个接口：

			public interface IActivityManager extends IInterface
			{
				......
			}
		这是对于IInterface接口的一个扩展接口，里面定义了ActivityManagerService所支持的一系列方法。

	2. Binder本地对象：

			public abstract class ActivityManagerNative extends Binder implements IActivityManager
		
		在这个例子中，ActivityManagerNative充当了Binder本地对象的角色，它继承自Binder类并实现了IActivityManager接口。

	3. Binder代理对象：

			class ActivityManagerProxy implements IActivityManager

		在此例中，Client端通过调用ActivityManagerNative的静态方法getDefault()可以得到其代理对象，即ActivityMangerProxy的实例，这个代理对象如上所说，实现了与Binder本地对象相同的IActivityManager接口，也即支持其所有的方法，但它除了对参数进行包装并传递外并没有做任何其他事情。

	我们以startActivity(...)方法为例，来看一下上述几个item是如何协同完成一次跨进程通信的。

	这里直接从Instrumentation类的execStartActivity(...)方法开始，之前还有一些与跨进程通信无关的步骤，在此不做累述。

		public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) 
		{
			......

			int result = ActivityManagerNative.getDefault()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
            checkStartActivityResult(result, intent);

			......
		}

	可以看到这里其实是调用了ActivityManagerNative.getDefault()中的startActivity(...)方法，见到熟人了，上面已经提到过这个getDefault方法会返回一个ActivityManagerNative的代理类，也就是之前所说的Binder代理对象，下面就点进去看看：

		static public IActivityManager getDefault() 
		{
        	return gDefault.get();
    	}

	简单到令人发指的一个方法，返回值是IActivityManager的实例，再看看gDefault是个什么鬼：

		private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() 
		{
	        protected IActivityManager create() 
			{
	            IBinder b = ServiceManager.getService("activity");
	            if (false)
				{
	                Log.v("ActivityManager", "default service binder = " + b);
	            }
	            IActivityManager am = asInterface(b);
	            if (false) 
				{
	                Log.v("ActivityManager", "default service = " + am);
	            }
	            return am;
	        }
    	};

	gDefault是一个全局静态的单例变量，当gDefault().get()被调用时，上面的这个create()方法会被回调，里面可以看到先是通过"activity"这个key从ServiceManager中获取了一个IBinder对象，这里具体的应该是**ActivityManagerService(AMS)**对象，AMS是Android中最核心的Service，主要负责系统中四大组件的启动、切换、调度及应用进程的管理和调度等工作。最后在得到AMS实例之后将其给到返回IActivityManager实例的asInterface(...)方法中，下面一个个来看：

	**先看IBinder b = ServiceManager.getService("activity");**（这里做一个**标记**，第一个方法走下去的会比较深，等下介绍完之后会再次回到这里接上，继续看第二个方法）

		public static IBinder getService(String name) 
		{
	        try {
	            IBinder service = sCache.get(name);
	            if (service != null) {
	                return service;
	            } else {
	                return getIServiceManager().getService(name);
	            }
	        } catch (RemoteException e) {
	            Log.e(TAG, "error in getService", e);
	        }

	        return null;
    	}

	这是getService的第一步，很明确，先去cache中找这个传入的name(这里是"activity")，如果找到的话就直接返回，否则调用getIServiceManager().getService(name);接着看getIServiceManager()方法：

		private static IServiceManager getIServiceManager() {
        if (sServiceManager != null) {
            return sServiceManager;
        }

        // Find the service manager
        sServiceManager = ServiceManagerNative.asInterface(BinderInternal.getContextObject());
        return sServiceManager;
    	}

	这是ServiceManager中的一个私有静态方法，它是通过ServiceManagerNative.asInterface(...)这个方法获得一个实现了IServiceManager接口的本地或远程Binder对象，来看看sdk中对传入参数BinderInternal.getContextObject()的描述：

		/**
     	* Return the global "context object" of the system.  This is usually
     	* an implementation of IServiceManager, which you can use to find
     	* other services.
     	*/
    	public static final native IBinder getContextObject();

	可以看出这是一个native的方法，最终返回了一个IServiceManager的实例，通常来讲这是系统级别的实现。到这里就可以看出，之前出现在ServiceManager.getService("activity")这一句中的ServiceManager只不过是一个类似工具类的家伙，或者理解为一个代理类，里面提供了一系列供上层调用的静态接口，真正管理系统Service的其实是getIServiceManager()方法返回的IServiceManager的实例。

	ServiceManagerNative.asInterface(BinderInternal.getContextObject());这是getIServiceManager()最终的返回值，这个过程跟我们接下来要分析的获取Binder对象的过程一致，这里不做累述，只需要知道经过上述过程我们获得了系统真正的ServiceManager。

	再往下就很好理解了，拿到了系统管理Service的manager类，那么通过"activity"这个key自然是能够得到ActivityManagerService(AMS)实例。但我们再往深一步，看看AMS是何时何地通过"activity"这个key被注册到系统的ServiceManager中的。翻看AMS的源码，发现有这么一个public方法：

		 public void setSystemProcess()
		 {
			 ......

			 ServiceManager.addService(Context.ACTIVITY_SERVICE, this, true);

			 ......
		 }

		 其中，Context.ACTIVITY_SERVICE = "activity"; 这个静态变量声明在Context类中。

	明显的，当这个方法被调用时，AMS就会被注册到ServiceManager中。这个方法是在SystemServer class中被调用，上源码：

		private void startBootstrapServices() 
		{
			.......

			// Activity manager runs the show.
	        mActivityManagerService = mSystemServiceManager.startService(
	                ActivityManagerService.Lifecycle.class).getService();
	        mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
	        mActivityManagerService.setInstaller(installer);

			......

			// Set up the Application instance for the system process and get started.
        	mActivityManagerService.setSystemProcess();

			......
		}

	这里就是AMS被创建并且初始化的地方，startBootstrapServices()是一个私有方法，那么再看看它被SystemServer调用的地方：

		/**
     	* The main entry point from zygote.
     	*/
	    public static void main(String[] args) 
		{
	        new SystemServer().run();
	    }

		private void run() 
		{
			......

			startBootstrapServices();

			......
		}

	很清楚了，startBootstrapServices()是在run()方法中被调到的，而这个run()这就是Zygote进程启动系统服务进程SystemServer的主入口。所以系统服务进程SystemServer被启动的时候，AMS就被初始化并加入到了ServiceManager中。

	至此，我们就通过ServiceManager.getService("activity")；方法获得了一个AMS的实例，**现在跳回到之前黑体字标记的地方**， 继续看第二个方法**asInterface(IBinder obj)**：

		/**
     	* Cast a Binder object into an activity manager interface, generating
     	* a proxy if needed.
     	*/
	    static public IActivityManager asInterface(IBinder obj)
		{
	        if (obj == null) {
	            return null;
	        }
	        IActivityManager in =
	            (IActivityManager)obj.queryLocalInterface(descriptor);
	        if (in != null) {
	            return in;
	        }
	
	        return new ActivityManagerProxy(obj);
	    }

	[*注：String descriptor = "android.app.IActivityManager"；是定义在IActivityManager中的，类似一个key*]

	这个方法是真正给Client端提供Binder代理对象的，首先看函数的参数IBinder类型的obj，这里我们传入的便是刚刚拿到的AMS实例，接下俩分为**两种**情况：先是通过queryLocalInterface方法判断Client和Server是否运行在同一个进程中，如果是，即obj属于本地对象，则直接返回AMS对象本身；否则就代表Server端处在另一个进程中，所以这里会new一个代理对象ActivityManagerProxy并返回。

	接下来就进入到了ActivityManagerProxy中，如上所述，这个类其实就是一个代理，自己并没有实现任何的功能，完全是调用构造函数中传入的IBinder对象，也即一个远程的AMS对象。

	看一下构造函数：

		class ActivityManagerProxy implements IActivityManager
		{
		    public ActivityManagerProxy(IBinder remote)
		    {
		        mRemote = remote;
		    }
		
		    public IBinder asBinder()
		    {
		        return mRemote;
		    }
	
			......
		}

	可以看到代理对象中全局保存了传入的远程binder对像为mRemote。

	下面继续看这个代理中的startActivity(...)方法：

	    public int startActivity(IApplicationThread caller, String 	callingPackage, Intent intent, String resolvedType, IBinder resultTo, 	String resultWho, int requestCode, int startFlags, ProfilerInfo 	profilerInfo, Bundle options) throws RemoteException 
		{
	        Parcel data = Parcel.obtain();
	        Parcel reply = Parcel.obtain();
	        data.writeInterfaceToken(IActivityManager.descriptor);
	        data.writeStrongBinder(caller != null ? caller.asBinder() : null);
	        data.writeString(callingPackage);
	        intent.writeToParcel(data, 0);
	        data.writeString(resolvedType);
	        data.writeStrongBinder(resultTo);
	        data.writeString(resultWho);
	        data.writeInt(requestCode);
	        data.writeInt(startFlags);
	        if (profilerInfo != null) {
	            data.writeInt(1);
	            profilerInfo.writeToParcel(data, Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
	        } else {
	            data.writeInt(0);
	        }
	        if (options != null) {
	            data.writeInt(1);
	            options.writeToParcel(data, 0);
	        } else {
	            data.writeInt(0);
	        }
	        mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
	        reply.readException();
	        int result = reply.readInt();
	        reply.recycle();
	        data.recycle();
	        return result;
    	}

	这里贴出了这个方法里的全部代码，可以看到方法中创造了两个新对象：data和reply。然后先是根据传入参数对data进行了包装和处理，再下来传递式调用之前提到的Binder远程对象mRemote的transact(...)方法，参数flags为0时方法阻塞，等待Server端对应方法返回后继续执行。参数flags为FLAG_ONEWAY时立即返回。这里要注意的是传入的reply对象此时还是一个新得到的对象[*注：Parcel.obtain()方法不一定会new一个新对象，而是会先去看缓存中有没有可用的*]，而对其的赋值是发生在Server端的，这个之后会看到。调用完transact(...)方法后，proxy会从刚才的reply对象中取出赋值并作为结果返回给Client端。

	以上就是ActivityManagerProxy这个Binder代理对象在这个流程所做的所有事情，实际上就是帮忙整理参数并调用真正远程Binder对象的transact(...)方法，再往下就是远程Server端的任务了，先看transact(...)方法：

		/**
		* Default implementation rewinds the parcels and calls onTransact. On
		* the remote side, transact calls into the binder to do the IPC.
		*/
    	public final boolean transact(int code, Parcel data, Parcel reply, int flags) throws RemoteException 
		{
	        if (false) Log.v("Binder", "Transact: " + code + " to " + this);
	
	        if (data != null) {
	            data.setDataPosition(0);
	        }

	        boolean r = onTransact(code, data, reply, flags);

	        if (reply != null) {
	            reply.setDataPosition(0);
	        }
	        return r;
    	}

	这里可以看到，先是将包含输入参数的Parcel对象data的读写位置置为0，然后调用Binder的onTransact(...)方法：

		@Override
    	public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
            throws RemoteException 
		{
			switch (code) 
			{
				case START_ACTIVITY_TRANSACTION:
				{
					data.enforceInterface(IActivityManager.descriptor);
		            IBinder b = data.readStrongBinder();
		            IApplicationThread app = ApplicationThreadNative.asInterface(b);
		            String callingPackage = data.readString();
		            Intent intent = Intent.CREATOR.createFromParcel(data);
		            String resolvedType = data.readString();
		            IBinder resultTo = data.readStrongBinder();
		            String resultWho = data.readString();
		            int requestCode = data.readInt();
		            int startFlags = data.readInt();
		            ProfilerInfo profilerInfo = data.readInt() != 0 ? ProfilerInfo.CREATOR.createFromParcel(data) : null;
		            Bundle options = data.readInt() != 0 ? Bundle.CREATOR.createFromParcel(data) : null;

		            int result = startActivity(app, callingPackage, intent, resolvedType,
		                    resultTo, resultWho, requestCode, startFlags, profilerInfo, options);

		            reply.writeNoException();
		            reply.writeInt(result);

		            return true;
        		}

				......

			}

			......

		}

	在这个方法中，通过传入的code进入相应的代码段，上面贴出来的就是startActivity相关的代码，可以看到其中调用了startActivity(...)方法，需要注意的是执行这段代码的已经是之前得到的mRemote即AMS实例对象，所以这里自然也是调用ActivityManagerService.startActivity(...)方法，也就是远程Server端Binder对象的中的对应方法，接下来便是AMS在远程进行startActivity的过程，这里就先省略了。

	再往下，拿到result后便将result写入reply对象中，然后如之前所说，ActivityManagerProxy.transact()从阻塞状态返回，Client端得到reply中的返回参数，处理并继续执行。

	至此，一次完整的Binder-IPC跨进程通信就完成了，可以看到，Binder机制维持了Client进程的transact()的调用传递给Server端transact()以及相应的调用返回的传递过程。注意参数flags为FLAG_ONEWAY时指定通信为“单向”的，这样整个远程调用就成为了异步的——Client的transact()会很快返回，不需要等待Server端的方法调用完成（甚至是开始）。

- **参考文章**：
	
	http://www.jianshu.com/p/af2993526daf
	
	http://blog.csdn.net/luoshengyang/article/details/6689748

	http://www.cnblogs.com/everhad/p/6246551.html