---
title: Runtime 调用Process.waitfor导致的阻塞问题
comments: true
date: 2019-06-16 09:34:20
tags:
    - Runtime
---

### 1. 关于Runtime类的小知识
> **1. Runtime.getRuntime()可以取得当前JVM的运行时环境，这也是在Java中唯一一个得到运行时环境的方法**
> **2. Runtime中的exit方法是退出JVM**

### 2. Runtime的几个重要的重载方法
<!-- more -->
|方法名|作用|
|--|--|
|exec(String command);|在单独的进程中执行指定的字符串命令。  |
|exec(String command, String[] envp) |  在指定环境的单独进程中执行指定的字符串命令。|
| exec(String[] cmdarray, String[] envp, File dir)  | 在指定环境和工作目录的独立进程中执行指定的命令和变量  |
|exec(String command, String[] envp, File dir) |在有指定环境和工作目录的独立进程中执行指定的字符串命令。 |
Runtime类的重要的方法还有很多，简单列举几个
>1.exit(int status)：终止当前正在运行的 Java 虚拟机
>2.freeMemory()： 返回 Java 虚拟机中的空闲内存量。
>3.load(String filename)： 加载作为动态库的指定文件名。
>4.loadLibrary(String libname)： 加载具有指定库名的动态库。
### 3. Runtime的使用方式

**3.1 错误的使用exitValue()**
```
  public static void main(String[] args) throws IOException {
        String command = "ping www.baidu.com";
        Process process = Runtime.getRuntime().exec(command);
        int i = process.exitValue();
        System.out.println("字进程退出值："+i);
    }
```
**输出：**
```
Exception in thread "main" java.lang.IllegalThreadStateException: process has not exited
	at java.lang.ProcessImpl.exitValue(ProcessImpl.java:443)
	at com.lirong.think.runtime.ProcessUtils.main(ProcessUtils.java:26)
```
**原因：**
> exitValue()方法是非阻塞的，在调用这个方法时cmd命令并没有返回所以引起异常。阻塞形式的方法是waitFor，它会一直等待外部命令执行完毕，然后返回执行的结果。

**修改后的版本：**
```
  public static void main(String[] args) throws IOException {
        String command = "javac";
        Process process = Runtime.getRuntime().exec(command);
        process.waitFor();
        process.destroy();
        int i = process.exitValue();
        System.out.println("字进程退出值："+i);
    }
```
**此版本已然可以正常运行，但当主线程和子线程有很多交互的时候还是会出问题，会出现卡死的情况。**

#### 4. 卡死原因
 >**主进程中调用Runtime.exec会创建一个子进程，用于执行cmd命令。子进程创建后会和主进程分别独立运行。
 因为主进程需要等待脚本执行完成，然后对命令返回值或输出进行处理，所以这里主进程调用Process.waitfor等待子进程完成。
 运行此cmd命令可以知道：子进程执行过程就是打印信息。主进程中可以通过Process.getInputStream和Process.getErrorStream获取并处理。
 这时候子进程不断向主进程发生数据，而主进程调用Process.waitfor后已挂起。当前子进程和主进程之间的缓冲区塞满后，子进程不能继续写数据，然后也会挂起。
这样子进程等待主进程读取数据，主进程等待子进程结束，两个进程相互等待，最终导致死锁。**

#### 5. 解决方案
不断的读取消耗缓冲区的数据，以至子进程不会挂起，下面是具体代码：

```
/**
 * @author lirong
 * @desc CMD命令测试
 * @date 2019/06/13 20:50
 */
@Slf4j
public class ProcessUtils {
    public static void main(String[] args) throws IOException, InterruptedException {
        String command = "ping www.baidu.com";
        Process process = Runtime.getRuntime().exec(command);
        readStreamInfo(process.getInputStream(), process.getErrorStream());
        int exit = process.waitFor();
        process.destroy();
        if (exit == 0) {
            log.debug("子进程正常完成");
        } else {
            log.debug("子进程异常结束");
        }
    }

    /**
     * 读取RunTime.exec运行子进程的输入流 和 异常流
     * @param inputStreams 输入流
     */
    public static void readStreamInfo(InputStream... inputStreams){
        ExecutorService executorService = Executors.newFixedThreadPool(inputStreams.length);
        for (InputStream in : inputStreams) {
            executorService.execute(new MyThread (in));
        }
        executorService.shutdown();
    }
}

/**
 * @author lirong
 * @desc
 * @date 2019/06/13 21:25
 */
@Slf4j
public class MyThread implements Runnable {

    private InputStream in;
    public MyThread(InputStream in){
        this.in = in;
    }

    @Override
    public void run() {
        try{
            BufferedReader br = new BufferedReader(new InputStreamReader(in, "GBK"));
            String line = null;
            while((line = br.readLine())!=null){
                log.debug(" inputStream: " + line);
            }
        }catch (IOException e){
            e.printStackTrace();
        }finally {
            try {
                in.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}
```
写到这里大家以为都结束了哇，并没有，哈哈哈，真实的生成环境总能给你带来很多神奇的问题，Runtime不仅可以直接调用CMD执行命令，还可以调用其他.exe程序执行命令。所以纵使你读取了缓冲区的数据，你的程序依然可能会被卡死，因为有可能你的缓冲区根本就没有数据，而是你的.exe程序卡主了。嗯，所以为你以防万一，你还需要设置超时。

### 6. Runtime最优雅的调用方式
```
/**
 * @author lirong
 * @desc
 * @date 2019/06/13 20:50
 */
@Slf4j
public class ProcessUtils {

   /**
     * @param timeout 超时时长
     * @param fileDir 所运行程序路径
     * @param command 程序所要执行的命令
     * 运行一个外部命令，返回状态.若超过指定的超时时间，抛出TimeoutException
     */
    public static int executeProcess(final long timeout, File fileDir, final String[] command)
            throws IOException, InterruptedException, TimeoutException {
        Process process = Runtime.getRuntime().exec(command, null, fileDir);
        Worker worker = new Worker(process);
        worker.start();
        try {
            worker.join(timeout);
            if (worker.exit != null){
                return worker.exit;
            } else{
                throw new TimeoutException();
            }
        } catch (InterruptedException ex) {
            worker.interrupt();
            Thread.currentThread().interrupt();
            throw ex;
        }
        finally {
            process.destroy();
        }
    }
    
    private static class Worker extends Thread {
        private final Process process;
        private Integer exit;

        private Worker(Process process) {
            this.process = process;
        }

        @Override
        public void run() {
            InputStream errorStream = null;
            InputStream inputStream = null;
            try {
                errorStream = process.getErrorStream();
                inputStream = process.getInputStream();
                readStreamInfo(errorStream, inputStream);
                exit = process.waitFor();
                process.destroy();
                if (exit == 0) {
                    log.debug("子进程正常完成");
                } else {
                    log.debug("子进程异常结束");
                }
            } catch (InterruptedException ignore) {
                return;
            }
        }
    }

    /**
     * 读取RunTime.exec运行子进程的输入流 和 异常流
     * @param inputStreams 输入流
     */
    public static void readStreamInfo(InputStream... inputStreams){
        ExecutorService executorService = Executors.newFixedThreadPool(inputStreams.length);
        for (InputStream in : inputStreams) {
            executorService.execute(new MyThread(in));
        }
        executorService.shutdown();
    }
}
```
