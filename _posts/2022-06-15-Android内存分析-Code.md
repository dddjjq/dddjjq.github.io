# Android内存分析-meminfo流程以及部分内存的组成

## 一、引言

Android内存优化属于性能优化中非常重要的一环，对应用工程师来说，很多指标都是通过口口相传的方式，显得并不清晰，甚至有些描述是错误的，今天我们通过源码分析一下内存的dump过程与部分内存构成。

## 二、meminfo

![img](https://github.com/dddjjq/dddjjq.github.io/raw/main/_posts/image/2022-06-15/1.png)

在Android中，我们可以通过dumpsys meminfo packagename的方式拿到对应应用的具体内存信息，那么这个meminfo是怎么统计的呢？

我们先理一下代码流程

### 1、cmd入口

dumpsys这条指令由dumpsys.cpp执行，这一段我们粗略的说一下，并不是核心

frameworks/native/cmds/dumpsys/dumpsys.cpp

首先是main函数，main函数中会解析我们命令行中带的参数，然后进入startDumpThread函数：

~~~c++
通过binder获取远端引用，这里在AMS中注册，后面提到
sp<IBinder> service = sm_->checkService(serviceName);
...
调用binder方法
status_t err = service->dump(remote_end.get(), args);
~~~

核心的流程是获取Binder句柄，然后调用dump方法。

### 2、Service

在AMS中有这样一段代码：

~~~java
    public void setSystemProcess() {
        try {
          ...
            ServiceManager.addService("meminfo", new MemBinder(this), /* allowIsolated= */ false,
                    DUMP_FLAG_PRIORITY_HIGH);
            ServiceManager.addService("gfxinfo", new GraphicsBinder(this));
            ServiceManager.addService("dbinfo", new DbBinder(this));
            mAppProfiler.setCpuInfoService();
            ServiceManager.addService("permission", new PermissionController(this));
            ServiceManager.addService("processinfo", new ProcessInfoService(this));
            ServiceManager.addService("cacheinfo", new CacheBinder(this));
          ....
        }
    }
~~~

这里是向ServiceManager注册相关的服务，假如我们dump meminfo，那么前面提到的binder句柄就是MemBinder的代理，remote方法也是调用这里：

~~~java
@Override
public void dump(FileDescriptor fd, PrintWriter pw, String[] args, boolean asProto) {
		mActivityManagerService.dumpApplicationMemoryUsage(
      fd, pw, "  ", args, false, null, asProto);
}
~~~

MemBinder的dump方法调用到AMS的dumpApplicationMemoryUsage方法中，最终调用Debug的getMemoryInfo方法:

### 3、getMemoryInfo

frameworks/base/core/java/android/os/Debug.java

getMemoryInfo是一个native方法，进入：

frameworks/base/core/jni/android_os_Debug.cpp

~~~c++
static jboolean android_os_Debug_getDirtyPagesPid(JNIEnv *env, jobject clazz,
        jint pid, jobject object)
{
    bool foundSwapPss;
    stats_t stats[_NUM_HEAP];
    memset(&stats, 0, sizeof(stats));
	//读取进程信息，并存到stats中
    if (!load_maps(pid, stats, &foundSwapPss)) {
        return JNI_FALSE;
    }
  
	//Graphics内存读取
    struct graphics_memory_pss graphics_mem;
    if (read_memtrack_memory(pid, &graphics_mem) == 0) {
        stats[HEAP_GRAPHICS].pss = graphics_mem.graphics;
        stats[HEAP_GRAPHICS].privateDirty = graphics_mem.graphics;
        stats[HEAP_GRAPHICS].rss = graphics_mem.graphics;
        stats[HEAP_GL].pss = graphics_mem.gl;
        stats[HEAP_GL].privateDirty = graphics_mem.gl;
        stats[HEAP_GL].rss = graphics_mem.gl;
        stats[HEAP_OTHER_MEMTRACK].pss = graphics_mem.other;
        stats[HEAP_OTHER_MEMTRACK].privateDirty = graphics_mem.other;
        stats[HEAP_OTHER_MEMTRACK].rss = graphics_mem.other;
    }

    for (int i=_NUM_CORE_HEAP; i<_NUM_EXCLUSIVE_HEAP; i++) {
        stats[HEAP_UNKNOWN].pss += stats[i].pss;
        stats[HEAP_UNKNOWN].swappablePss += stats[i].swappablePss;
        stats[HEAP_UNKNOWN].rss += stats[i].rss;
        stats[HEAP_UNKNOWN].privateDirty += stats[i].privateDirty;
        stats[HEAP_UNKNOWN].sharedDirty += stats[i].sharedDirty;
        stats[HEAP_UNKNOWN].privateClean += stats[i].privateClean;
        stats[HEAP_UNKNOWN].sharedClean += stats[i].sharedClean;
        stats[HEAP_UNKNOWN].swappedOut += stats[i].swappedOut;
        stats[HEAP_UNKNOWN].swappedOutPss += stats[i].swappedOutPss;
    }

  	//对Java层对象赋值
    for (int i=0; i<_NUM_CORE_HEAP; i++) {
        env->SetIntField(object, stat_fields[i].pss_field, stats[i].pss);
        env->SetIntField(object, stat_fields[i].pssSwappable_field, stats[i].swappablePss);
        env->SetIntField(object, stat_fields[i].rss_field, stats[i].rss);
        env->SetIntField(object, stat_fields[i].privateDirty_field, stats[i].privateDirty);
        env->SetIntField(object, stat_fields[i].sharedDirty_field, stats[i].sharedDirty);
        env->SetIntField(object, stat_fields[i].privateClean_field, stats[i].privateClean);
        env->SetIntField(object, stat_fields[i].sharedClean_field, stats[i].sharedClean);
        env->SetIntField(object, stat_fields[i].swappedOut_field, stats[i].swappedOut);
        env->SetIntField(object, stat_fields[i].swappedOutPss_field, stats[i].swappedOutPss);
    }


    env->SetBooleanField(object, hasSwappedOutPss_field, foundSwapPss);
    jintArray otherIntArray = (jintArray)env->GetObjectField(object, otherStats_field);

    jint* otherArray = (jint*)env->GetPrimitiveArrayCritical(otherIntArray, 0);
    if (otherArray == NULL) {
        return JNI_FALSE;
    }

  	//对应Java层对相关属性的获取
    //stats的属性在一维数组中展开，最终存在Java层的otherStats中
    int j=0;
    for (int i=_NUM_CORE_HEAP; i<_NUM_HEAP; i++) {
        otherArray[j++] = stats[i].pss;
        otherArray[j++] = stats[i].swappablePss;
        otherArray[j++] = stats[i].rss;
        otherArray[j++] = stats[i].privateDirty;
        otherArray[j++] = stats[i].sharedDirty;
        otherArray[j++] = stats[i].privateClean;
        otherArray[j++] = stats[i].sharedClean;
        otherArray[j++] = stats[i].swappedOut;
        otherArray[j++] = stats[i].swappedOutPss;
    }

    env->ReleasePrimitiveArrayCritical(otherIntArray, otherArray, 0);
    return JNI_TRUE;
}

~~~

读取的核心逻辑在load_maps里面：

~~~c++
static bool load_maps(int pid, stats_t* stats, bool* foundSwapPss)
{
    *foundSwapPss = false;
    uint64_t prev_end = 0;
    int prev_heap = HEAP_UNKNOWN;
	//核心！ 读取进程的smaps文件，并从中读取相关的信息 
    std::string smaps_path = base::StringPrintf("/proc/%d/smaps", pid);
    auto vma_scan = [&](const meminfo::Vma& vma) {
        int which_heap = HEAP_UNKNOWN;
        int sub_heap = HEAP_UNKNOWN;
        bool is_swappable = false;
        std::string name;
        if (base::EndsWith(vma.name, " (deleted)")) {
            name = vma.name.substr(0, vma.name.size() - strlen(" (deleted)"));
        } else {
            name = vma.name;
        }

        uint32_t namesz = name.size();
        if (base::StartsWith(name, "[heap]")) {
            which_heap = HEAP_NATIVE;
        } else if (base::StartsWith(name, "[anon:libc_malloc]")) {
            which_heap = HEAP_NATIVE;
        } else if (base::StartsWith(name, "[anon:scudo:")) {
            which_heap = HEAP_NATIVE;
        } else if (base::StartsWith(name, "[anon:GWP-ASan")) {
            which_heap = HEAP_NATIVE;
        } else if (base::StartsWith(name, "[stack")) {
            which_heap = HEAP_STACK;
        } else if (base::StartsWith(name, "[anon:stack_and_tls:")) {
            which_heap = HEAP_STACK;
        } else if (base::EndsWith(name, ".so")) {
            which_heap = HEAP_SO;
            is_swappable = true;
        } else if (base::EndsWith(name, ".jar")) {
            which_heap = HEAP_JAR;
            is_swappable = true;
        } else if (base::EndsWith(name, ".apk")) {
            which_heap = HEAP_APK;
            is_swappable = true;
        } else if (base::EndsWith(name, ".ttf")) {
            which_heap = HEAP_TTF;
            is_swappable = true;
        } else if ((base::EndsWith(name, ".odex")) ||
                (namesz > 4 && strstr(name.c_str(), ".dex") != nullptr)) {
            which_heap = HEAP_DEX;
            sub_heap = HEAP_DEX_APP_DEX;
            is_swappable = true;
        } else if (base::EndsWith(name, ".vdex")) {
            which_heap = HEAP_DEX;
            // Handle system@framework@boot and system/framework/boot|apex
            if ((strstr(name.c_str(), "@boot") != nullptr) ||
                    (strstr(name.c_str(), "/boot") != nullptr) ||
                    (strstr(name.c_str(), "/apex") != nullptr)) {
                sub_heap = HEAP_DEX_BOOT_VDEX;
            } else {
                sub_heap = HEAP_DEX_APP_VDEX;
            }
            is_swappable = true;
        }
        ...
        stats[which_heap].pss += usage.pss;
        stats[which_heap].swappablePss += swapable_pss;
        stats[which_heap].rss += usage.rss;
        stats[which_heap].privateDirty += usage.private_dirty;
        stats[which_heap].sharedDirty += usage.shared_dirty;
        stats[which_heap].privateClean += usage.private_clean;
        stats[which_heap].sharedClean += usage.shared_clean;
        stats[which_heap].swappedOut += usage.swap;
        stats[which_heap].swappedOutPss += usage.swap_pss;
        ...
    };

    return meminfo::ForEachVmaFromFile(smaps_path, vma_scan);
}

~~~

主要流程是读取进程的smaps文件，然后转换成相应的数据结构，读取相应的值作为上层Java层对象的值，这里面每项对应的相关内存通过将结构体展开，并存入响应的数组index中，这样Java层在读取的时候，相关项在枚举中方的index一致即可。

### 4、Code部分组成

在最终形成的meminfo中，有一项Code，这一项是通过Debug中的 getSummaryCode得到的：
~~~java
public int getSummaryCode() {
    return getOtherPrivate(OTHER_SO)
      + getOtherPrivate(OTHER_JAR)
      + getOtherPrivate(OTHER_APK)
      + getOtherPrivate(OTHER_TTF)
      + getOtherPrivate(OTHER_DEX)
      + getOtherPrivate(OTHER_OAT)
      + getOtherPrivate(OTHER_DALVIK_OTHER_ZYGOTE_CODE_CACHE)
      + getOtherPrivate(OTHER_DALVIK_OTHER_APP_CODE_CACHE);
}
~~~
~~~java
public int getOtherPrivate(int which) {
    return getOtherPrivateClean(which) + getOtherPrivateDirty(which);
}
public int getOtherPrivateClean(int which) {
	return otherStats[which * NUM_CATEGORIES + OFFSET_PRIVATE_CLEAN];
}
~~~
可以看到，获取code部分是通过获取PrivateClean以及PrivateDirty部分的之和得到的，heap类型包含OTHER_SO、OTHER_JAR、OTHER_TTF、OTHER_DEX等。以OTHER_JAR为例，获取otherStats的index为OTHER_JAR*NUM_CATEGORIES+OFFSET_PRIVATE_CLEAN也就是7x9+5，这里的7对应的是android_os_Debug枚举中的HEAP_JAR，OFFSET_PRIVATE_CLEAN对应的是PrivateClean在stats结构体中的index，其他的部分以此类推。

### 5、Code分析
了解了Code的组成，我们对于这部分的优化也可以更加的得心应手，我们可以参考load_maps的实现，将smaps文件pull出来，然后写python脚本将Code的组成部分的PrivateClean与PrivateDirty部分打印出来（可以只打印相对比较大的部分），这样就可以直观的看到具体是哪一部分占用的内存过多，再进行针对性的分析。（具体可以见[dump_smaps.py](https://github.com/dddjjq/ScriptTools/blob/main/Android/Memory/dump_smaps.py)）

### 6、Graphics分析

Graphics比较特殊，并不是从smaps文件中得到的。
从前面的android_os_Debug_getDirtyPagesPid函数中，我们可以看到其实这里是调用read_memtrack_memory来读取graphics相关的内存：
~~~c++
/*
 * Retrieves the graphics memory that is unaccounted for in /proc/pid/smaps.
 */
static int read_memtrack_memory(int pid, struct graphics_memory_pss* graphics_mem)
{
    ...
    int err = read_memtrack_memory(p, pid, graphics_mem);
    memtrack_proc_destroy(p);
    return err;
}
~~~
从注释可以看到，这部分其实是读取那些不在smaps中计数的内存，这里我们不在具体分析，可以参考[Memtrack hal](https://www.cnblogs.com/pyjetson/p/14769359.html)。Graphics这部分内存是GPU所占用的内存，包含EGL mtrack以及GL mtrack（GPU内存还包含一个Gfx dev，但是这些内存可以在smaps中读取到，以/dev/kgsl-3d0开头的就是这部分内存）。EGL mtrack/GL mtrack这部分内存是平台相关的，在高通平台上面有这样的映射关系：     
**EGL mtrack => /sys/class/kgsl/kgsl/proc/[pid]/imported_mem  
GL mtrack  => /sys/class/kgsl/kgsl/proc/[pid]/gpumem_unmapped**

读取这些文件可以发现，其实里面只有数值，对于GPU的内存分配不是特别了解，暂时只能到此为止。

### 7、其他

像Java Heap以及Native Heap，市面上已经有相对完善的分析工具，我们这里暂时就不再深入。

## 三、结论

起因是公司的项目中从某个版本开始Code部分增长50M左右，但并没有行之有效的分析工具，所以从源码方面做一些分析。

我们这里主要是分析了一下meminfo的组成结构，且主要是分析Code部分的组成，其他部分的比较类似，一些具体的概念以及其他命令可以在参考部分获取。

meminfo主要是通过读取进程的smaps文件，然后统计里面的相关信息得到。Code部分包含so、jar、apk、jit cache等，我们可以通过Python脚本的形式来做统计，可以更加直观。

由于Code这部分有点众说纷纭，不能很好地区分到底是哪些部分，所以我们这里从系统的统计方式出发，以这种形式来达到我们最终优化的目的。

## 四、参考

[1、Gityuan Android内存分析命令](http://gityuan.com/2016/01/02/memory-analysis-command/)

[2、android相机场景下整机内存分析](https://blog.csdn.net/buhui912/article/details/115909242)

[3、Memtrack hal](https://www.cnblogs.com/pyjetson/p/14769359.html)

[4、ELC: How much memory are applications really using?](https://lwn.net/Articles/230975/)