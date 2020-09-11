# 虚幻集成puerts攻略

（虚幻版本4.25，puerts代码: 2020.9.10更新）

## monolith方式（输出是一个.a）

1. 准备个好代理，后面要下载不少东西
2. 开WSL（如果是linux直接跳过此步）
3. apt-get install git
4. 新建个目录，然后
    ```
        git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
    ```
5. 将depot_tools加到环境变量Path里，直接改windows的就行
6. 执行gclient
7. 在新建个目录（假设叫v8_root）
    ```
        git clone https://github.com/v8/v8.git
    ```
8. cd 到v8目录，然后 git checkout v8的某个稳定版分支（这里checkout了8.5.210.20分支）
9.  回到v8_root，从 https://github.com/mldbai/v8-cross-build 下载.gclient和.gclient_entries
10. 打开.gclient，最底部增加 target_os = ['android'] 保存
11. 命令行执行 gclient sync，然后就是漫长的等待，要下载好多东西
12. 中间如果有python报错，那么sudo apt install python下，如果安装找不到路径，那么 apt-get update 下
13. 执行 tools/dev/v8gen.py gen arm64.release -vv，如果有报错找不到glib，安之：sudo apt-get install libglib2.0-dev
14. 将v8/out.gn/arm64.release/args.gn 中的内容替换为

    ```
        # 所有选项可通过tools/dev/v8gen.py list输出
        use_custom_libcxx = false
        is_component_build = false
        is_official_build = true
        is_debug = false
        #symbol_level 不能是2，否则会报错，怀疑跟clang版本有关，可以通过修改build.gn屏蔽报错
        symbol_level = 0
        target_cpu = "arm64"
        v8_target_cpu = "arm64"
        target_os = "android"
        use_goma = false
        v8_android_log_stdout = true
        v8_enable_i18n_support = false
        v8_static_library = true
        # 开启此模式后，最后会生成一个大lib，里面什么都有，如果不开启，那么需自行合并.a
        v8_monolithic = true
        strip_debug_info = true
        android64_ndk_api_level = 24
        android_ndk_major_version = 21
        # 虚幻使用的ndk路径（PS：虚幻ubt默认使用ndk中自带的clang，路径写死到代码，无法通过配置更改）
        android_ndk_root = "/mnt/e/NVPACK/android-ndk-r21b-linux-x86_64/android-ndk-r21b"
        android_ndk_version = "r21b"
        # 虚幻使用的ndk对应的clang路径，这里需注意，如果不指定clang相关版本，那么v8默认使用clang11，编出来的.a虚幻无法link
        clang_base_path = "/mnt/e/NVPACK/android-ndk-r21b-linux-x86_64/android-ndk-r21b/toolchains/llvm/prebuilt/linux-x86_64"
        clang_version = "9.0.8"
        # 由于4.25 ndk中自带的clang版本比较老，所以这里会有不少warning，需开启下面选项，否则编译无法进行
        treat_warnings_as_errors = false
        clang_use_chrome_plugins = false
        use_thin_lto = false
        # 开启v8_monolithic后，snapshot信息会打入到最终lib里，无需在手动输入snapshot信息
        v8_use_external_startup_data = false
    ```

15. 终于可以编了！在v8目录执行 ninja -C out.gn/arm64.release
16. build完成后，在\v8\out.gn\arm64.release\obj 里可以找到所需的libv8_monolith.a文件，将其考到插件ThirdParty\Library\V8\Android\arm64-release目录下
17. 将v8\include代码拷贝到插件ThirdParty\Include\v8\8.5.210.20目录下
18. 如果插件是原始代码，那么需要修改JsEnv.Build.cs
    ```
        // 修改版本号
        //PublicIncludePaths.AddRange(new string[] { Path.Combine(HeaderPath, "v8", "7.4.288") });
        PublicIncludePaths.AddRange(new string[] { Path.Combine(HeaderPath, "v8", "8.5.210.20") });
    ```

    ```
        // 安卓平台下
        // 注掉此代码
        //PublicAdditionalLibraries.Add(Path.Combine(ARM64LibPath, "libinspector.a"));
        //PublicAdditionalLibraries.Add(Path.Combine(ARM64LibPath, "libv8_base.a"));
        //PublicAdditionalLibraries.Add(Path.Combine(ARM64LibPath, "libv8_libbase.a"));
        //PublicAdditionalLibraries.Add(Path.Combine(ARM64LibPath, "libv8_libplatform.a"));
        //PublicAdditionalLibraries.Add(Path.Combine(ARM64LibPath, "libv8_libsampler.a"));
        //PublicAdditionalLibraries.Add(Path.Combine(ARM64LibPath, "libv8_external_snapshot.a"));

        // 修改android平台加载的lib名
        PublicAdditionalLibraries.Add(Path.Combine(ARM64LibPath, "libv8_monolith.a"));

        // 需加此宏，否则启动崩溃
        PublicDefinitions.Add("V8_COMPRESS_POINTERS");
    ```


19. 继续修改源码，打开JsEnv.cpp
    ```
        // 注掉下面，因为v8版本比插件高，以及打的是monolith方式，所以不需要手动设置外部snapshot数据， V8JSEngine.cpp里也有，都需要注掉
        // 初始化Isolate和DefaultContext
        //v8::V8::SetNativesDataBlob(NativesBlob.get());
        //v8::V8::SetSnapshotDataBlob(SnapshotBlob.get());

        ...

        // 下面代码可以顺手加上，这样v8报错还能看到log
        MainIsolate->SetFatalErrorHandler([](const char* location, const char* message) {
            UE_LOG(LogTemp, Error, TEXT("FatalErrorCallback: location: %s, message: %s"), ANSI_TO_TCHAR(location), ANSI_TO_TCHAR(message));
        });
        MainIsolate->SetOOMErrorHandler([](const char* location, bool is_heap_oom) {
            UE_LOG(LogTemp, Error, TEXT("OOMErrorCallback: location: %s, is_heap_oom: %d"), ANSI_TO_TCHAR(location), is_heap_oom?1:0);
        });
        v8::V8::SetDcheckErrorHandler([](const char* file, int line, const char* message) {
            UE_LOG(LogTemp, Error, TEXT("DcheckErrorCallback: file: %s, line: %d, message: %s"), ANSI_TO_TCHAR(file), line, ANSI_TO_TCHAR(message));
        });
    ```



20. 因为插件指定的文件夹路径都是写死的，所以可以根据项目调整代码达到项目本身的路径需求

## 非monolith方式（插件原始方式，未走通，所以下面步骤仅供参考）

1. 1-13步跟上面一样
2. 将v8/out.gn/arm64.release/args.gn 中的内容替换为

    ```
        use_custom_libcxx = false
        is_component_build = false
        is_official_build = true
        is_debug = false
        symbol_level = 0
        target_cpu = "arm64"
        v8_target_cpu = "arm64"
        target_os = "android"
        use_goma = false
        v8_android_log_stdout = true
        v8_enable_i18n_support = false
        v8_static_library = true
        # 非monolith模式主要就是不能开此选项
        #v8_monolithic = true
        strip_debug_info = true
        android64_ndk_api_level = 24
        android_ndk_major_version = 21
        android_ndk_root = "/mnt/e/NVPACK/android-ndk-r21b-linux-x86_64/android-ndk-r21b"
        android_ndk_version = "r21b"
        clang_base_path = "/mnt/e/NVPACK/android-ndk-r21b-linux-x86_64/android-ndk-r21b/toolchains/llvm/prebuilt/linux-x86_64"
        clang_version = "9.0.8"
        treat_warnings_as_errors = false
        clang_use_chrome_plugins = false
        use_thin_lto = false
        # 非v8_monolithic需打开此选项
        v8_use_external_startup_data = true
    ```
3. 在v8目录执行 ninja -C out.gn/arm64.release
4. 编完后，在v8路径新建merge-libs.sh文件，内容如下：
    ```
        cd out.gn/arm64.release/obj
        rm -rf libs
        mkdir libs
        cd libs

        # 合并各种lib
        ar -rcsD libv8_base.a ../v8_base_without_compiler/*.o
        ar -rcsD libv8_base.a ../v8_compiler/*.o
        ar -rcsD libv8_base.a ../v8_init/*.o
        ar -rcsD libv8_base.a ../v8_initializers/*.o

        ar -rcsD libv8_libbase.a ../v8_libbase/*.o
        ar -rcsD libv8_libplatform.a ../v8_libplatform/*.o

        ar -rcsD libv8_external_snapshot.a ../v8_snapshot/*.o

        ar -rcsD libinspector.a ../src/inspector/inspector/*.o
        ar -rcsD libv8_libsampler.a ../v8_libsampler/*.o
    ```
5. 执行此shell，然后将out.gn/arm64.release/obj/libs 下的所有lib拷贝到插件目录
6. 将v8\include代码拷贝到插件ThirdParty\Include\v8\8.5.210.20目录下
7. 如果是高版本v8，那么需注掉下面代码：
    ```
        //v8::V8::SetNativesDataBlob(NativesBlob.get());
    ```


参考：

    https://cstsinghua.github.io/2018/04/04/%E6%9E%84%E5%BB%BAv8%20engine%E6%8C%87%E5%8D%97/

