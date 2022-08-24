---
layout: post
tags: android
---

# simpleperf耗时分析

simpleperf是ndk包下的一个性能分析工具，专门用于分析ndk代码。

simpleperf有两种工作方式，一种是可执行文件的方式，一种是脚本的方式。

下载ndk包，里面自带simpleperf。

/Users/xm210408/Library/Android/sdk/ndk/21.1.6352462/simpleperf
在simpleperf目录下，有很多的脚本可供使用

```sh
$ ls | grep "\.py"
__init__.py
annotate.py
api_profiler.py
app_profiler.py
binary_cache_builder.py
debug_unwind_reporter.py
pprof_proto_generator.py
profile_pb2.py
report.py
report_html.py
report_sample.py
run_simpleperf_on_device.py
run_simpleperf_without_usb_connection.py
simpleperf_report_lib.py
utils.py
```

## 介绍

https://developer.android.com/ndk/guides/simpleperf

## 输出性能数据

记录设备上app运行过程，在本机pc上生成性能数据perf.data。用到的脚本是app_profiler.py。

```sh
kangzhou-mac:simpleperf xm210408$ python app_profiler.py --help
usage: app_profiler.py [-h]
                       (-p APP | -np NATIVE_PROGRAM | -cmd CMD | --pid PID [PID ...] | --tid TID [TID ...] | --system_wide)
                       [--compile_java_code] [-a ACTIVITY | -t TEST]
                       [-r RECORD_OPTIONS] [-lib NATIVE_LIB_DIR]
                       [-o PERF_DATA_PATH] [-nb] [--ndk_path NDK_PATH]
                       [--disable_adb_root] [--log {debug,info,warning}]

app_profiler.py: Record cpu profiling data of an android app or native program.

    It downloads simpleperf on device, uses it to collect profiling data on the selected app,
    and pulls profiling data and related binaries on host.

optional arguments:
  -h, --help            show this help message and exit

Select profiling target:
  -p APP, --app APP     Profile an Android app, given the package name. Like
                        `-p com.example.android.myapp`.
  -np NATIVE_PROGRAM, --native_program NATIVE_PROGRAM
                        Profile a native program running on the Android
                        device. Like `-np surfaceflinger`.
  -cmd CMD              Profile running a command on the Android device. Like
                        `-cmd "pm -l"`.
  --pid PID [PID ...]   Profile native processes running on device given their
                        process ids.
  --tid TID [TID ...]   Profile native threads running on device given their
                        thread ids.
  --system_wide         Profile system wide.

Extra options for profiling an app:
  --compile_java_code   Used with -p. On Android N and Android O, we need to
                        compile Java code into native instructions to profile
                        Java code. Android O also needs wrap.sh in the apk to
                        use the native instructions.
  -a ACTIVITY, --activity ACTIVITY
                        Used with -p. Profile the launch time of an activity
                        in an Android app. The app will be started or
                        restarted to run the activity. Like `-a
                        .MainActivity`.
  -t TEST, --test TEST  Used with -p. Profile the launch time of an
                        instrumentation test in an Android app. The app will
                        be started or restarted to run the instrumentation
                        test. Like `-t test_class_name`.

Select recording options:
  -r RECORD_OPTIONS, --record_options RECORD_OPTIONS
                        Set recording options for `simpleperf record` command.
                        Use `run_simpleperf_on_device.py record -h` to see all
                        accepted options. Default is "-e task-clock:u -f 1000
                        -g --duration 10".
  -lib NATIVE_LIB_DIR, --native_lib_dir NATIVE_LIB_DIR
                        When profiling an Android app containing native
                        libraries, the native libraries are usually stripped
                        and lake of symbols and debug information to provide
                        good profiling result. By using -lib, you tell
                        app_profiler.py the path storing unstripped native
                        libraries, and app_profiler.py will search all shared
                        libraries with suffix .so in the directory. Then the
                        native libraries will be downloaded on device and
                        collected in build_cache.
  -o PERF_DATA_PATH, --perf_data_path PERF_DATA_PATH
                        The path to store profiling data. Default is
                        perf.data.
  -nb, --skip_collect_binaries
                        By default we collect binaries used in profiling data
                        from device to binary_cache directory. It can be used
                        to annotate source code and disassembly. This option
                        skips it.

Other options:
  --ndk_path NDK_PATH   Set the path of a ndk release. app_profiler.py needs
                        some tools in ndk, like readelf.
  --disable_adb_root    Force adb to run in non root mode. By default,
                        app_profiler.py will try to switch to root mode to be
                        able to profile released Android apps.
  --log {debug,info,warning}
                        set log level
```



```
python app_profiler.py -p com.tencent.olhct2soLibrary -lib /Users/xm210408/program2/olhct_sdk/app/build/intermediates/cmake/debug/obj/arm64-v8a/
```

-p simpleperf.example.cpp是手机中android项目的进程，-lib libdir是ndk编译后的so所在的目录，运行python脚本需要python3的环境。

## 生成报告

解析性能数据文件perf.data，生成分析报告。

```
python report_html.py --add_source_code --source_dirs /Users/xm210408/program2/olhct_sdk/app/src/main/cpp --add_disassembly -o xxx.html
```

--source_dirs ../demo是源码cpp所在目录。
