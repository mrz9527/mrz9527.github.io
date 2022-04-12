# simpleperf

simpleperf是ndk包下的一个性能分析工具，专门用于分析ndk。

simpleperf有两种工作方式，一种是可执行文件的方式，一种是脚本的方式。

下载ndk包，里面自带simpleperf。

/Users/xm210408/Library/Android/sdk/ndk/21.1.6352462/simpleperf

准备工作

```
simpleperf输出性能数据和报告的命令，主要是在pc端运行。
simpleperf用到了adb，需要有adb环境。
手机连接pc机。
执行app_profiler.py脚本，按照提示，手动启动手机上相应app，自动输出性能数据
执行report_html.py脚本，输出报告。

```

## 输出性能数据

记录设备上app运行过程，在本机pc上生成性能数据perf.data。用到的脚本是app_profiler.py。

```
python app_profiler.py -p simpleperf.example.cpp -lib /Users/xm210408/programs/extras/simpleperf/demo/SimpleperfExampleCpp/app/build/intermediates/cmake/debug/obj/arm64-v8a/

python app_profiler.py -p com.tencent.olhct2soLibrary -lib /Users/xm210408/program2/olhct_sdk/app/build/intermediates/cmake/debug/obj/arm64-v8a/
```

-p simpleperf.example.cpp是手机中android项目的进程，-lib libdir是ndk编译的so所在的目录，运行python脚本需要python3的环境。

## 生成报告

解析性能数据文件perf.data，生成分析报告。

report_html.py在simpleperf目录下。--source_dirs ../demo是源码cpp所在目录。

```
python report_html.py --add_source_code --source_dirs ../demo --add_disassembly

python report_html.py --add_source_code --source_dirs /Users/xm210408/program2/olhct_sdk/app/src/main/cpp --add_disassembly -o xxx.html
```

## 其他功能

在simpleperf目录下，还有很多其他的脚本可供使用。

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

## 官方资料

```sh
https://developer.android.com/ndk/guides/simpleperf
```

