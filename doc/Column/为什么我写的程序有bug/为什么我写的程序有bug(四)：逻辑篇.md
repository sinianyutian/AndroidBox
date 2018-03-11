# 前言

逻辑性错误也是出现bug的重灾区，有很多是因为逻辑性比较复杂，这个倒是可以理解。但是，很多时候出现的问题查了半天最后真想给自己一巴掌。人傻没办法，自己折腾自己。因为这个问题实在太弱智了。

# 一、语法使用不当

先说个例子，伪代码如下：

	for(A a: AList){
		if(a.get()<0){
			//do something when < 0
			//continue；
		}
		if(a.get()<10){
			//do something when < 10
			//return；
		}
		//do something when > 10
	}
前段时间就写了一个这样的傻逼bug,找了老半天。我本来是希望 when <10 之后应该continue的，在写第一个when<0的时候头脑还是很清晰的。但是呢，当写第二个的时候就用四肢写代码了，习惯性的打了个return。很明显，还有许多该做的事情都没有做就跳出循环了。

像这样的错误还有什么时候容易犯呢？比如：

+ "!" 非判断的时候，容易搞反了。
+ 三目运算符，写错位置。记住，true在前，false在后
+ return 和 continue 搞混了或者习惯性写成了return
+ if的条件判断里面有&&或者||，两者搞混或者是后面修改的时候没有看清楚
+ 多个连续的 if elseif else 判断中的判断条件有交叉

# 二、前后不一致

先说个列子，伪代码如下：

	if(request.type == Response.Code.DATA_REQUEST){
		// do something
	}

这里本来的意思针对请求的类型不同进行处理，但是我们在进行对比的时候，用Request的类型和Response的类型进行比较，显然存在问题。这种问题在进行ReView代码的时候还常常想当然的认为是OK的。

再说个例子，伪代码如下：

	/*
     * 将 accountEntity 的数据转为 dBAccountEntity，方便进行数据库操作
     */
	private void convertToDb( AccountEntity  accountEntity, DBAccountEntity dBAccountEntity ){
		dBAccountEntity.setName(accountEntity.getName());
        dbAccountEntity.setAge(dbAccountEntity.getAge());//错误姿势一
		accountEntity.setTel(accountEntity.getTel());//错误姿势一
	}
我们在进行数据库存储数据时，通常需要进行数据实体的转换。由于我们通常还需进行反方向的转换，所以这里一不小心在“copy"或者直接写的时候搞反了，埋下了祸根。

像这类的问题还有？

+ SQLite的字段设置为了unique的，但是insert的时候有重复。这个异常内部捕获了，返回了-1。这里尤其是"_id";


# 三、判断条件出现漏洞

先上伪代码：

    private void onSuccess(){
		if (a>=10){
			//do somthing when a>=10
    	}
	}
	
    private void onFailed(){
		if (a>10){
			//do somthing when a>=10
    	}
	}

上述我们可以看到，本来的意思是当a>=10时，在成功或者失败的回调要处理不一样的逻辑。但是我们在进行if判断时候写错了。这里举例的比较简单，尤其是当我们有复杂条件的判断时，需要注意判断条件是否符合预期。

# 四、线程问题

### 1、多个线程操作一个对象的问题
先举个例子，依然是伪代码：

	// Thread-1
	for(Account account: accountList){
		// do something
	}
	// Thread-2
    private removeAccount(Account account){
		// accountList.remove(account);
	}

这里的线程问题相对比较明显，有问题还会报Exception,应该是大家都知晓的问题，有些甚至作为代码规范的一条。但是同类的问题却很多，在逻辑相对比较复杂的情况下，容易挖坑。

### 2、同一个线程的顺序问题

再说个比较隐晦点的，伪代码如下：
	
	 private class AccountServiceConnection implements ServiceConnection {

        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
			//主线程回调
            accountService = IPushService.Stub.asInterface(service);
            Log.i(TAG, "AccountService connected.:" + (Looper.getMainLooper() == Looper.myLooper()));
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            accountService = null;
        }
    }
	
    public void getInfo(){
		if(accountService == null){
			waitBind();//等待绑定成功、然后跨进程获取数据
		}
		Info info = pushService.aidlGetInfo();
	}
    
发现这个问题的现象是程序刚启动时随机出现几次ANR,随后发现外部在主线程调用了getInfo()方法，但是调用该方法之前，serveice还没有绑定成功。根据getInfo方法里面的逻辑，他在此处wait，随后将主线程阻塞。而此时绑定Service的回调onServiceConnected（）也是在主线程回调的。前面已经将主线程阻塞了，那么这里永远也无法回调回来。回调不回来，那getInfo()里面就一直wait。最终导致了ANR,这种问题在非主线程也要注意。

### 3、多个线程的顺序问题
举个列子，伪代码如下：

	
    private void onAccountLoginSuccess(){
		mHandler.post(new Runnable() {
             @Override
             public void run() {
                updateOnSuccess()
             }
        });
		Account newAccount = getAccount();
	}

	//Thread-1
	private synchronize void updateOnSuccess(){
		mAccount.setLoginSuccess(true)
		
	}
	//Thread-2
	private synchronize Account getAccount(){
		return mAccount;
	}

这里和1可能有些重复，但我还是想单独区分开来。因为平时我们可能用同步锁处理了线程安全问题，但是因为我们无法保证updateOnSuccess（）和getAccount（）方法哪个先调用哪个后调用。

#  五、数据持久化以及缓存处理问题
举个例子，上伪代码：

	private int getAccountID(){
		if(mAccount!=0){
			return mAccountID;//内存中的缓存
		}
		return AccountDao.getAccountID();//从数据库读取
	}

	private void onLoginFailed(){
		//假设登录失败需要清除ID重新注册
		AccountDao.deleteAccountID()；
		toLoginActivity();
	}

	private void loginAccount(){
		int accountID;
		if(getAccountID()!=0){
			//有id,则自动登录
			login（accountID）;
		}else{
			//手动登录
		}
	}

类似于上述的一个逻辑，本来发现登录返回失败，原因是accountID不对，这个时候我们可能会删除accountID，然后重新输入账号密码登录。但是，我们在清除accountID时只清除了数据库，没有清除缓存，再次登录的时候用的缓存的值。这样就会导致程序陷入了死循环。所以在进行存储操作时，需要考虑好同步的问题。


# 改善建议

+ 遵循Java编程的设计原则
+ 尽量将复杂的问题简单化
+ 做好关注点分离，分成业务模块，减少耦合，内部的事情内部处理，对外提供接口
+ 借用成熟开源框架的设计思想
+ 写单元测试


