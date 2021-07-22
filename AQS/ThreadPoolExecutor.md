1.几个状态


    //用一个Integer的大小来表,高三位代表线程池的状态,低位代表线程个数
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    //线程个数掩码位数,并不是所有平台int类型是32位,所以准确说是具体平台下Integer的二进制位数-3后的剩余位数才是线程的个数
    private static final int COUNT_BITS = Integer.SIZE - 3;
    //线程最大个数 00011111111111111111111111111111
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
    // runState is stored in the high-order bits
    //高3位:11100000000000000000000000000000
    private static final int RUNNING    = -1 << COUNT_BITS; //接受新任务并且处理阻塞队列里的任务;
    //高3位:00000000000000000000000000000000
    private static final int SHUTDOWN   =  0 << COUNT_BITS; //拒绝新任务但是处理阻塞队列里的任务;
    //高3位:00100000000000000000000000000000
    private static final int STOP       =  1 << COUNT_BITS; //拒绝新任务并且抛弃阻塞队列里的任务,同时会中断正在处理的任务;
    //高3位:01000000000000000000000000000000
    private static final int TIDYING    =  2 << COUNT_BITS;//所有任务都执行完(包含阻塞队列里面任务)当前线程池活动线程为 0,将要调用terminated();
    //高3位:01100000000000000000000000000000
    private static final int TERMINATED =  3 << COUNT_BITS;//终止状态,terminated()调用完成以后的状态;

    // Packing and unpacking ctl
    //获取高三位 线程池运行状态
    private static int runStateOf(int c)     { return c & ~CAPACITY; }
    //获取低29位 线程个数
    private static int workerCountOf(int c)  { return c & CAPACITY; }
    //计算ctl新值，线程状态 与 线程个数
    private static int ctlOf(int rs, int wc) { return rs | wc; }

2.execute(Runnable command) 方法执行


    public void execute(Runnable command) {
            //1)判断任务是否为null
            if (command == null)
                throw new NullPointerException();
            //获取当前线程池的状态和线程数量
            int c = ctl.get();
            //线程个数小于corePoolSize,小于则尝试开启一个新的线程，并且加入到工作线程的集合中
            if (workerCountOf(c) < corePoolSize) {
                if (addWorker(command, true))
                    return;
                c = ctl.get();
            }
            //如果线程池处于运行,并且把线程任务添加到队列中成功
            if (isRunning(c) && workQueue.offer(command)) {
                int recheck = ctl.get();
                //二次校验,判断线程池是否允许状态,并且从队列中删除,然后执行拒绝策略
                if (!isRunning(recheck) && remove(command))
                    reject(command);
                //二次校验判断工作线程数量为0,尝试开启一个新的线程    
                else if (workerCountOf(recheck) == 0)
                    addWorker(null, false);
            }
            //如果上面分支失败了,则尝试开启一个新的线程,如果开启线程任务失败，执行拒绝策略，
            //1.如果线程池状态处于非运行状态，直接执行拒绝策略
            //2.如果队列满了，入队失败，则尝试开启一个新的线程
            //例如配置为corePoolSize=1，workQueue=1，maximumPoolSize=2时，当线程的核心数满了，队列满了，wc >= maximumPoolSize 为false
            
            else if (!addWorker(command, false))
                reject(command);
        }
3.addWorker(Runnable firstTask, boolean core) 方法执行
    
    //firstTask 待执行线程任务，core是否为核心线程
    private boolean addWorker(Runnable firstTask, boolean core) {
            retry:
            for (;;) {
            //获取线程池个数和状态
                int c = ctl.get();
            //获取线程池状态    
                int rs = runStateOf(c);
            //检查队列为空的情况，只有再线程池状态时shutdown之后的状态(包含shutdown)
            //并且workerCountOf(recheck) == 0 才有可能进入这个分支
                if (rs >= SHUTDOWN &&
                    ! (rs == SHUTDOWN &&
                       firstTask == null &&
                       ! workQueue.isEmpty()))
                    return false;
    
                for (;;) {
                //获取工作线程个数
                    int wc = workerCountOf(c);
                //如果工作线程大于是否超出限制，超出返回false
                    if (wc >= CAPACITY ||
                        wc >= (core ? corePoolSize : maximumPoolSize))
                        return false;
                // cas增加线程个数，同时只能有一个成功
                    if (compareAndIncrementWorkerCount(c))
                        break retry;
                //cas 失败，再次获取线程池个数和状态
                    c = ctl.get();  // Re-read ctl
                //再次判断线程池状态是否有变化，如果发生了变化，则最外层循环continue执行
                    if (runStateOf(c) != rs)
                        continue retry;
                    // else CAS failed due to workerCount change; retry inner loop
                }
            }
    
            boolean workerStarted = false;
            boolean workerAdded = false;
            Worker w = null;
            try {
            //new 一个Worker
                w = new Worker(firstTask);
                final Thread t = w.thread;
                if (t != null) {
                    final ReentrantLock mainLock = this.mainLock;
                    //加独占锁，为了workers同步，因为可能多个线程调用了线程池的execute方法
                    mainLock.lock();
                    try {
                        //重新检查线程池状态，防止再获取锁之前 调用shuntdown
                        int rs = runStateOf(ctl.get());
                        if (rs < SHUTDOWN ||
                            (rs == SHUTDOWN && firstTask == null)) {
                            //检测线程是否时就绪状态
                            if (t.isAlive()) // precheck that t is startable
                                throw new IllegalThreadStateException();
                            //添加线程到工作线程集合中
                            workers.add(w);
                            int s = workers.size();
                            if (s > largestPoolSize)
                                largestPoolSize = s;
                            workerAdded = true;
                        }
                    } finally {
                        mainLock.unlock();
                    }
                    //添加成功，开始启动线程
                    if (workerAdded) {
                        t.start();
                        workerStarted = true;
                    }
                }
            } finally {
            //如果工作线程开始失败，加入失败
                if (! workerStarted)
                    addWorkerFailed(w);
            }
            return workerStarted;
        }
        

4.runWorker(Worker w) 具体的执行任务内容的方法


    final void runWorker(Worker w) {
            Thread wt = Thread.currentThread();
            Runnable task = w.firstTask;
            w.firstTask = null;
            w.unlock(); // allow interrupts
            boolean completedAbruptly = true;
            try {
                while (task != null || (task = getTask()) != null) {
                    w.lock();
                    // If pool is stopping, ensure thread is interrupted;
                    // if not, ensure thread is not interrupted.  This
                    // requires a recheck in second case to deal with
                    // shutdownNow race while clearing interrupt
                    if ((runStateAtLeast(ctl.get(), STOP) ||
                         (Thread.interrupted() &&
                          runStateAtLeast(ctl.get(), STOP))) &&
                        !wt.isInterrupted())
                        wt.interrupt();
                    try {
                        beforeExecute(wt, task);
                        Throwable thrown = null;
                        try {
                            task.run();
                        } catch (RuntimeException x) {
                            thrown = x; throw x;
                        } catch (Error x) {
                            thrown = x; throw x;
                        } catch (Throwable x) {
                            thrown = x; throw new Error(x);
                        } finally {
                            afterExecute(task, thrown);
                        }
                    } finally {
                        task = null;
                        w.completedTasks++;
                        w.unlock();
                    }
                }
                completedAbruptly = false;
            } finally {
                processWorkerExit(w, completedAbruptly);
            }
        }


