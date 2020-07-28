-----


# java中常见的死锁以及解决方法代码
在java中我们常常使用加锁机制来确保线程安全，但是如果过度使用加锁，则可能导致锁顺序死锁。同样，我们使用线程池和信号量来限制对资源的使用，但是这些被限制的行为可能会导致资源死锁。java应用程序无法从死锁中恢复过来，因此设计时一定要排序那些可能导致死锁出现的条件。

# 1.一个最简单的死锁案例
当一个线程永远地持有一个锁，并且其他线程都尝试获得这个锁时，那么它们将永远被阻塞。在线程A持有锁L并想获得锁M的同时，线程B持有锁M并尝试获得锁L，那么这两个线程将永远地等待下去。这种就是最简答的死锁形式（或者叫做"抱死"）。

# 2.锁顺序死锁

![image9fd1888f2a8b6ced.png](https://wx1.sbimg.cn/2020/06/19/image9fd1888f2a8b6ced.png)

如图：leftRight和rightLeft这两个方法分别获得left锁和right锁。如果一个线程调用了leftRight，而另一个线程调用了rightLeft，并且这两个线程的操作是交互执行，那么它们就会发生死锁。

死锁的原因就是两个线程试图以不同的顺序来获得相同的锁。所以，如果所有的线程以固定的顺序来获得锁，那么在程序中就不会出现锁顺序死锁的问题。

## 2.1.动态的锁顺序死锁

我以一个经典的转账案例来进行说明，我们知道转账就是将资金从一个账户转入另一个账户。在开始转账之前，首先需要获得这两个账户对象得锁，以确保通过原子方式来更新两个账户中的余额，同时又不破坏一些不变形条件，例如 账户的余额不能为负数。

所以写出的代码如下：

	//动态的锁的顺序死锁
	public class DynamicOrderDeadlock {
	     
	    public static void transferMoney(Account fromAccount,Account toAccount,int amount,int from_index,int to_index) throws Exception {
	        System.out.println("账户 "+ from_index+"~和账户~"+to_index+" ~请求锁");
	         
	        synchronized (fromAccount) {
	            System.out.println("    账户 >>>"+from_index+" <<<获得锁");
	            synchronized (toAccount) {
	                System.out.println("          账户   "+from_index+" & "+to_index+"都获得锁");
	                if (fromAccount.compareTo(amount) < 0) {
	                    throw new Exception();
	                }else {
	                    fromAccount.debit(amount);
	                    toAccount.credit(amount);
	                }
	            }
	        }
	    } 
	    static class Account {
	        private int balance = 100000;//这里假设每个人账户里面初始化的钱
	        private final int accNo;
	        private static final AtomicInteger sequence = new AtomicInteger();
	         
	        public Account() {
	            accNo = sequence.incrementAndGet();
	        }
	         
	        void debit(int m) throws InterruptedException {
	            Thread.sleep(5);//模拟操作时间
	            balance = balance + m;
	        }
	         
	        void credit(int m) throws InterruptedException {
	            Thread.sleep(5);//模拟操作时间
	            balance = balance - m;
	        } 
	         
	        int getBalance() {
	            return balance;
	        }
	         
	        int getAccNo() {
	            return accNo;
	        }
	         
	        public int compareTo(int money) {
	            if (balance > money) {
	                return 1;
	            }else if (balance < money) {
	                return -1;
	            }else {
	                return 0;
	            }
	        }
	    }
	}

	public class DemonstrateDeadLock {
		private static final int NUM_THREADS = 5;
		private static final int NUM_ACCOUNTS = 5;
		private static final int NUM_ITERATIONS = 100000;
	
		public static void main(String[] args) {
			final Random rnd = new Random();
			final Account[] accounts = new Account[NUM_ACCOUNTS];
			
			for(int i = 0;i < accounts.length;i++) {
				accounts[i] = new Account();
			}
			
			
			class TransferThread extends Thread{
				@Override
				public void run() {
					for(int i = 0;i < NUM_ITERATIONS;i++) { 
						int fromAcct = rnd.nextInt(NUM_ACCOUNTS);
						int toAcct =rnd.nextInt(NUM_ACCOUNTS);
						int amount = rnd.nextInt(100);
						try {
							DynamicOrderDeadlock.transferMoney(accounts[fromAcct],accounts[toAcct], amount,fromAcct,toAcct);
								//InduceLockOrder.transferMoney(accounts[fromAcct],accounts[toAcct], amount);
							 //InduceLockOrder2.transferMoney(accounts[fromAcct],accounts[toAcct], amount);
						}catch (Exception e) {
							System.out.println("发生异常-------"+e);
						}
					}
				}
			}
			 
			for(int i = 0;i < NUM_THREADS;i++) {
				new TransferThread().start();
			}
		}
	
	}
打印结果如下：
注意：这里的结果是我把已经执行完的给删除后，只剩下导致死锁的请求.

![imaged5266d4da881676b.png](https://wx2.sbimg.cn/2020/06/19/imaged5266d4da881676b.png)

	//通过锁顺序来避免死锁
	public class InduceLockOrder {
	    private static final Object tieLock = new Object();
	 
	    public static void transferMoney(final Account fromAcct, final Account toAcct, final int amount)
	            throws Exception {
	 
	        class Helper {
	            public void transfer() throws Exception {
	                if (fromAcct.compareTo(amount) < 0) {
	                    throw new Exception();
	                } else {
	                    fromAcct.debit(amount);
	                    toAcct.credit(amount);
	                }
	            }
	        }
	 
	        int fromHash = System.identityHashCode(fromAcct);
	        int toHash = System.identityHashCode(toAcct);
	 
	        if (fromHash < toHash) {
	            synchronized (fromAcct) {
	                synchronized (toAcct) {
	                    new Helper().transfer();
	                }
	            }
	        } else if (fromHash > toHash) {
	            synchronized (toAcct) {
	                synchronized (fromAcct) {
	                    new Helper().transfer();
	                }
	            }
	        } else {
	            synchronized (tieLock) {
	                synchronized (fromAcct) {
	                    synchronized (toAcct) {
	                        new Helper().transfer();
	                    }
	                }
	            }
	        }
	    }
	 
	     
	    static class Account {
	        private int balance = 100000;
	        public Account() {
	 
	        }
	         
	        void debit(int m) throws InterruptedException {
	            Thread.sleep(5);
	            balance = balance + m;
	        }
	         
	        void credit(int m) throws InterruptedException {
	            Thread.sleep(5);
	            balance = balance - m;
	        } 
	         
	        int getBalance() {
	            return balance;
	        }
	        public int compareTo(int money) {
	            if (balance > money) {
	                return 1;
	            }else if (balance < money) {
	                return -1;
	            }else {
	                return 0;
	            }
	        }
	         
	    }
	 
	}

经过我测试，此方案可行，不会造成死锁。

**方案二**

在Account中包含一个唯一的，不可变的，值。比如说账号等。通过对这个值对对象进行排序。
具体代码如下

	public class InduceLockOrder2 {
	
		public static void transferMoney(final Account fromAcct, final Account toAcct, final int amount)
				throws Exception {
	
			class Helper {
				public void transfer() throws Exception {
					if (fromAcct.compareTo(amount) < 0) {
						throw new Exception();
					} else {
						fromAcct.debit(amount);
						toAcct.credit(amount);
					}
				}
			}
	
			int fromHash = fromAcct.getAccNo();
			int toHash = toAcct.getAccNo();
	
			if (fromHash < toHash) {
				synchronized (fromAcct) {
					synchronized (toAcct) {
						new Helper().transfer();
					}
				}
			} else if (fromHash > toHash) {
				synchronized (toAcct) {
					synchronized (fromAcct) {
						new Helper().transfer();
					}
				}
			} 
		}
	
		
		static class Account {
			private int balance = 100000;
			private final int accNo;
			private static final AtomicInteger sequence = new AtomicInteger();
			
			public Account() {
				accNo = sequence.incrementAndGet();
			}
			
			void debit(int m) throws InterruptedException {
				Thread.sleep(6);
				balance = balance + m;
			}
			
			void credit(int m) throws InterruptedException {
				Thread.sleep(6);
				balance = balance - m;
			} 
			
			int getBalance() {
				return balance;
			}
			
			int getAccNo() {
				return accNo;
			}
			public int compareTo(int money) {
				if (balance > money) {
					return 1;
				}else if (balance < money) {
					return -1;
				}else {
					return 0;
				}
			}
			
		}
	}
经过测试此方案也可行。

## 2.2在协作对象之间发生的死锁
如果在持有锁时调用某外部的方法，那么将出现活跃性问题。在这个外部方法中可能会获取其他的锁（这个可能产生死锁），或阻塞时间过长，导致其他线程无法及时获得当前持有的锁。

场景如下：Taxi代表出租车对象，包含当前位置和目的地。Dispatcher代表车队。当一个线程收到GPS更新事件时掉用setLocation,那么它首先更新出租车的位置，然后判断它是否到达目的地。如果已经到达，它会通知Dispatcher:它需要一个新的目的地。因为setLocation和notifyAvailable都是同步方法，因此掉用setLocation线程首先获取taxi的锁，然后在获取Dispatcher的锁。同样，掉用getImage的线程首先获取Dispatcher的锁，再获取每一个taxi的锁，这两个线程按照不同的顺序来获取锁，因此可能导致死锁。

能造成死锁的代码如下：

	//会发生死锁
	public class CooperatingDeadLock {
	
		// 坐标类
		class Point {
			private final int x;
			private final int y;
	
			public Point(int x, int y) {
				this.x = x;
				this.y = y;
			}
	
			public int getX() {
				return x;
			}
	
			public int getY() {
				return y;
			}
		}
	
		// 出租车类
		class Taxi {
			private Point location, destination;
			private final Dispatcher dispatcher;
			
			public Taxi(Dispatcher dispatcher) {
				this.dispatcher = dispatcher;
			}
			
			public synchronized Point getLocation() {
				return location;
			}
			
			
			public synchronized void setLocation(Point location) {
				this.location = location;
				if (location.equals(destination)) {
					dispatcher.notifyAvailable(this);
				}
			}
			
			
			public synchronized Point getDestination() {
				return destination;
			}
			
			public synchronized void setDestination(Point destination) {
				this.destination = destination;
			}
		}
	
		class Dispatcher {
			private final Set<Taxi> taxis;
			private final Set<Taxi> availableTaxis;
	
			public Dispatcher() {
				taxis = new HashSet<>();
				availableTaxis = new HashSet<>();
			}
			
			public synchronized void notifyAvailable(Taxi taxi) {
				availableTaxis.add(taxi);
			}
	
			public synchronized Image getImage() {
				Image image = new Image();
				for(Taxi t:taxis) {
					image.drawMarker(t.getLocation());
				}
				return image;
			}
		}
		
		class Image{
			public void drawMarker(Point p) {
				
			}
		}
	
	}
解决方案：使用开放掉用。
如果再调用某个方法时不需要持有锁，那么这种调用就被称为开放掉用。这种调用能有效的避免死锁，并且易于分析线程安全。

修改后的代码如下：

	//此方案不会造成死锁
	public class CooperatingNoDeadlock {
		// 坐标类
			class Point {
				private final int x;
				private final int y;
	
				public Point(int x, int y) {
					this.x = x;
					this.y = y;
				}
	
				public int getX() {
					return x;
				}
	
				public int getY() {
					return y;
				}
			}
	
			// 出租车类
			class Taxi {
				private Point location, destination;
				private final Dispatcher dispatcher;
				
				public Taxi(Dispatcher dispatcher) {
					this.dispatcher = dispatcher;
				}
				
				public synchronized Point getLocation() {
					return location;
				}
				
				
				public void setLocation(Point location) {
					boolean reachedDestination;
					synchronized (this) {
						this.location = location;
						reachedDestination = location.equals(destination);
					}
					if (reachedDestination) {
						dispatcher.notifyAvailable(this);
					}
				}
				
				
				public synchronized Point getDestination() {
					return destination;
				}
				
				public synchronized void setDestination(Point destination) {
					this.destination = destination;
				}
			}
	
			class Dispatcher {
				private final Set<Taxi> taxis;
				private final Set<Taxi> availableTaxis;
	
				public Dispatcher() {
					taxis = new HashSet<>();
					availableTaxis = new HashSet<>();
				}
				
				public synchronized void notifyAvailable(Taxi taxi) {
					availableTaxis.add(taxi);
				}
	
				public Image getImage() {
					Set<Taxi> copy;
					synchronized (this) {
						copy = new HashSet<>(taxis);
					}
					
					Image image = new Image();
					for(Taxi t:copy) {
						image.drawMarker(t.getLocation());
					}
					return image;
				}
				
				
			}
			
			class Image{
				public void drawMarker(Point p) {
					
				}
			}
	}
总结：活跃性故障是一个非常严重的问题，因为当出现活跃性故障时，除了终止应用程序之外没有其他任何机制可以帮助从这种故障中恢复过来。最常见的活跃性故障就是锁顺序死锁。在设计时应该避免产生顺序死锁：确保线程在获取多个锁时采用一直的顺序。最好的解决方案是在程序中始终使用开放掉用。这将大大减小需要同时持有多个锁的地方，也更容易发现这些地方。