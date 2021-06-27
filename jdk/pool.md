# ThreadPoolExecutor.Worker

ThreadPoolExecutor 的线程实例都是非后台运行模式

线程池的 Worker 对象继承 AbstractQueuedSynchronizer 和实现了 Runnable 接口，通过 AbstractQueuedSynchronizer 来实现当前工作对象的工作中/非工作中的切换，而实现 Runnable 接口在创建工作线程时，可以将本身对象传递给新建的线程对象这样就可以通过线程来调用本身的 Runnable 实现逻辑
