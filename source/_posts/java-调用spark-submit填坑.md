---
title: java 调用spark-submit填坑
date: 2018-08-24 16:50:31
tags: [java,spark,submit]
---

## 问题背景
>在开发过程中遇到了如下的问题:
>使用shell调用spark-submit提交程序时,不到一分钟跑完所有的流程.
>但是使用java调用shell进而调用spark-submit就会卡在parttion比较多的步骤,此问题我纠结了四天的时间.
>可是可的确是弥补了很多知识上的短板.
>排查过程如下:

<!--more-->

## 排查过程

### 集群问题
>首先怀疑的是集群问题,搜索度娘和谷哥有很多人告知是因为内存不够或者是分配给的内核不够,但是此类问题在过滤后台的日志来看是不存在的,

``` shell
 426.2 MB of 1 GB physical memory used; 2.2 GB of 2.1 GB virtual memory used
```

### 代码问题

>后来怀疑是代码问题，因为使用python去调用该程序也是没问题的．
>代码如下:

``` java
    public static String runShell(String command) throws Exception {

        String result = "";
        Process process = Runtime.getRuntime().exec(command);
        // 获取shell返回流
        BufferedInputStream bufferedInputStream = new BufferedInputStream(process.getInputStream());
        // 字符流转换字节流
        BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(bufferedInputStream));

        String line;
        while ((line = bufferedReader.readLine()) != null)
            result = line;
        try {
            process.waitFor();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            // 关闭输入流
            bufferedInputStream.close();
            bufferedReader.close();
            process.destroy();
        }

        return result;
    }
```
>首先来解释一下 waitFor() 方法的意义， waitFor() 表示当前 Process 所在的子线程处于等待状态，如有必要，一直要等到由该 Process 对象表示的进程已经终止，官网说如果我们在调用此方法时，如果不注意的话，很容易出现主线程阻塞， Process 也挂起的情况。这就是我遇到的问题.解决办法是，在调用 waitFor() 的时候， Process 需要向主线程汇报运行状况，所以要注意清空缓存区，即 InputStream 和 ErrorStream ，注意这里 InputStream 和 ErrorStream 都需要清空。
>但是上面的方式只是解决了标准输入流的读取，并且打印最后一行，并没有处理标准错误流，后来经过如下代码的测试，spark-submit提交的日志竟然打印在标准错误流中，所以也就解释了之前为什么会卡在partiion比较多的地方,因为该部分日志较多,达到了缓存区的上限,所以流程不能继续.
>错误的表象:

``` java
Container marked as failed: container_1535013730755_0026_01_000007 on host: rocket04.kylin.com. Exit status: 1. Diagnostics: Exception from container-launch.
Container id: container_1535013730755_0026_01_000007
Exit code: 1
Stack trace: ExitCodeException exitCode=1: 
	at org.apache.hadoop.util.Shell.runCommand(Shell.java:601)
	at org.apache.hadoop.util.Shell.run(Shell.java:504)
	at org.apache.hadoop.util.Shell$ShellCommandExecutor.execute(Shell.java:786)
	at org.apache.hadoop.yarn.server.nodemanager.DefaultContainerExecutor.launchContainer(DefaultContainerExecutor.java:213)
	at org.apache.hadoop.yarn.server.nodemanager.containermanager.launcher.ContainerLaunch.call(ContainerLaunch.java:302)
	at org.apache.hadoop.yarn.server.nodemanager.containermanager.launcher.ContainerLaunch.call(ContainerLaunch.java:82)
	at java.util.concurrent.FutureTask.run(FutureTask.java:262)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1145)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615)
	at java.lang.Thread.run(Thread.java:745)

```


>调研的现象如下:
``` java
ERROR>18/08/27 16:12:59 INFO cluster.YarnScheduler: Removed TaskSet 4.0, whose tasks have all completed, from pool 
ERROR>18/08/27 16:12:59 INFO scheduler.DAGScheduler: ResultStage 4 (saveAsTextFile at ReportNatureIncubationCustom.scala:265) finished in 0.635 s
ERROR>18/08/27 16:12:59 INFO scheduler.DAGScheduler: Job 2 finished: saveAsTextFile at ReportNatureIncubationCustom.scala:265, took 3.497685 s
ERROR>18/08/27 16:13:01 INFO spark.SparkContext: Invoking stop() from shutdown hook
```
>测试代码:

``` java
	public static String niceCallShell(String command) {
        String result = "";
        try {
            System.out.println("cmd start");
            Process p = Runtime.getRuntime().exec(command);  //调用Linux的相关命令
            new RunThread(p.getInputStream(), "INFO").start();
            new RunThread(p.getErrorStream(), "ERROR").start();
            int value = p.waitFor();
            if (value == 0)
                result = "complete";
            else
                result = "failed";
        } catch (IOException e) {
            e.printStackTrace();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        return result;
    }

    static class RunThread extends Thread {
        InputStream is;
        String printType;

        RunThread(InputStream is, String printType) {
            this.is = is;
            this.printType = printType;
        }

        public void run() {
            try {
                InputStreamReader isr = new InputStreamReader(is);
                BufferedReader br = new BufferedReader(isr);
                String line = null;
                while ((line = br.readLine()) != null)
                    System.out.println(printType + ">" + line);
            } catch (IOException ioe) {
                ioe.printStackTrace();
            }
        }
    }
```


## 解决方案

### 不改java代码
>该解决方案为把spark-submit的日志重定向到一个独立文件中,就不会发生与java交互的Input/Error,方案如下:

``` shell
master yarn-client $deployPath/tourism-data-1.0.jar ${reportId} > spark_cumstom_${reportId}.log 2>&1
```

### 改动java代码
>这部分实则就是改动下之前的测试代码,将标准输入和标准错误放到不同的线程中,然后分发日志到logback中.

### 比较与总结
>第一种方法相比较与第二种方法而言,对于后期排查问题而言,可以更加直观,可以在shell文件的同级目录下找到所有的执行log,便于排查问题.
>第二种则需要在系统级别的log中找到需要的信息,比较繁琐,如果数据较大,同时并发度很大,则嵌入的日志很大.但是相比较来说第二种,比较传统,也更接近java 的系统调用的使用习惯.当然也可返回shell调用成功的最后一行的flag,但是不方便排查问题.
>综上:最终选择了第一种方案.




## 写在后面

### 快捷排查
>怎么通过自带的UI来定位错误信息已经日志信息，已经各个节点上的日志信息预览？

``` shell
yarn application list
```
>通过以上的命令进而通过我们设定的应用名称从而定位到Tracking-URL，在保证你本地已配置好数据集群的ip和host配置关系以后，通过该地址可以找到以下界面:

![spark ui ][1]

>从下图定位到executor

![executors][2]

>下图为详细信息

![executors][3]

  [1]: /ITWO/assets/spark-task-ui.png
  [2]: /ITWO/assets/executors.png
  [3]: /ITWO/assets/error-log.png