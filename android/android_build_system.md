
## 参考
Android soong build系统介绍：https://www.jianshu.com/p/80013a768a45


## Android build系统发展履历

Android7.0以前：使用makefile来组织编译构建系统

Android7.0:为了改善make的build效率，将make工具变更为ninja，ninja依赖*.ninja,就像make依赖makefile一样。当时Android7.0工程里面既存了大量的makefile，所以`为了使用ninja`，同时`避免makefile大规模改造`，开发了kati将makefile翻译成*.ninja文件

Android8.0 改造makefile，引入Android.bp，build的过程还是使用ninja，目的是逐步替换掉make-makefile这一build系统
由于引入了bp文件，而最终的build过程依赖的还是.ninja文件，这需要一个转换过程，由此创建了Soong，负责将Android.bp转换为.ninja文件，同时完成`选择编译`、`解析配置`的功能，如通过编译开关进行选择性编译。

Android9.0 系统主体编译过程已经完全由Soong+Ninja+Kati控制，由Kati将Android.mk转换为.ninja;由Soong将Android.bp转换为.ninja,由Ninja读取.ninja文件，完成最终的build过程。

Soong也生成了一个androidmk程序，用来将Android.mk转换为Android.bp文件

## Android9.0 build启动过程

Android系统编译指令
``` sh
source build/envsetup.sh
lunch 
make -jn
```

---
**make命令执行**
envsetup.sh在build/envsetup.sh

envsetup.sh主要是用于初始化shell环境，使其支持android build系统的特有命令：`croot`、`mgrep`、`cgrep`、`m`、`mm`等，和build系统相关的主要是`m`、`mm`、`mmm`、`mma`、`ma`、`make`，尤其是`make`

当我们执行make，在envsetup.sh初始化后的shell环境中，执行的是function make函数

``` sh
function make()
{
    _wrap_build $(get_make_command "$@") "$@"
}
```

> 这里判断当前make是在build系统根目录下，则执行build/soong/soong_ui.bash --make-mode
注意：这里是echo，使用echo作为get_make_command返回值，我们调用的是make -j8，
此时$@就是-j8，经过shell展开处理后`_wrap_build $(get_make_command "$@") "$@"`展开成了
`build/soong/soong_ui.bash --make-mode -j8`


``` sh
# _wrap_build()调用是，调用的是_wrap_build(build/soong/soong_ui.bash --make-mode，-j8)
function _wrap_build()
{
    local start_time=$(date +"%s")
    "$@"
    local ret=$?
    ...
}
# 从调用的角度看，是get_make_command(-j8)
# 经过get_make_command()函数处理后，得到build/soong/soong_ui.bash --make-mode字符串
function get_make_command()
{
    # If we're in the top of an Android tree, use soong_ui.bash instead of make
    
    if [ -f build/soong/soong_ui.bash ]; then
        # Always use the real make if -C is passed in
        for arg in "$@"; do
            if [[ $arg == -C* ]]; then
                echo command make
                return
            fi
        done
        echo build/soong/soong_ui.bash --make-mode
    else
        echo command make
    fi
}

```

---
**soong_ui.bash执行**

> 使用`soong_build_go soong_ui android/soong/cmd/soong_ui`来生成soong_ui程序
然后启动`exec "$(getoutdir)/soong_ui" "$@"`,这里$@即是--make-mode -j8

soong_ui.bash在build/soong/soong_ui.bash

``` sh
# Save the current PWD for use in soong_ui
export ORIGINAL_PWD=${PWD}
export TOP=$(gettop)
source ${TOP}/build/soong/scripts/microfactory.bash

soong_build_go soong_ui android/soong/cmd/soong_ui

cd ${TOP}
exec "$(getoutdir)/soong_ui" "$@"
```

**soong_ui build**

> 这里`soong_build_go soong_ui android/soong/cmd/soong_ui`，其$1即是确定的bin名称(soong_ui)
而package_name为`android/soong`/cmd/soong_ui

经过下面的shell执行结果，即使用microfactory工具编译了`build/soong/cmd/soong_ui/main.go`，生成了soong_ui程序
注意：是先生成的microfactory_Linux程序，进而使用microfactory_Linux生成soong_ui程序

``` sh
# Arguments:
#  $1: name of the requested binary
#  $2: package name
function soong_build_go
{
    BUILDDIR=$(getoutdir) \
      SRCDIR=${TOP} \
      BLUEPRINTDIR=${TOP}/build/blueprint \
      EXTRA_ARGS="-pkg-path android/soong=${TOP}/build/soong" \
      build_go $@
}
source ${TOP}/build/blueprint/microfactory/microfactory.bash
```

build_go是位于microfactory.bash的shell函数，这里不再继续深入了(也尚未完全掌握)

---

**soong_ui执行**

build/soong/cmd/soong_ui/main.go

这里启动`build.FindSources`，遍历源文件目录树中的Android.mk、Android.bp、cleanspec.mk，将其存放在out/.module_paths
├── Android.bp.list
├── Android.mk.list
├── CleanSpec.mk.list
├── files.db
└── TEST_MAPPING.list

接着启动`build.Build`完成整个的build过程。

``` go
func main() {
    ...
	f := build.NewSourceFinder(buildCtx, config)
	defer f.Shutdown()
	build.FindSources(buildCtx, config, f)

	if os.Args[1] == "--dumpvar-mode" {
		dumpVar(buildCtx, config, os.Args[2:])
	} else if os.Args[1] == "--dumpvars-mode" {
		dumpVars(buildCtx, config, os.Args[2:])
	} else {
		toBuild := build.BuildAll
		if config.Checkbuild() {
			toBuild |= build.RunBuildTests
		}
		build.Build(buildCtx, config, toBuild)
	}
}
```

---
**阶段总结**

经过上述操作后，在out目录生成了如下文件，这里总结性说明如下：

├── Android.mk
├── build_date.txt
├── build.trace.gz
├── CleanSpec.mk
├── .lock
├── .lock_build.trace.gz
├── .lock_soong.log
├── microfactory_Linux  用来生成soong_ui的程序
├── .microfactory_Linux_hash：microfactory_Linux build过程中判断hash变化从而判断是否启动重新编译
├── .microfactory_Linux_intermediates：生成microfactory_Linux用的中间文件
│   ├── github.com-google-blueprint-microfactory
│   │   └── github.com
│   │       └── google
│   │           └── blueprint
│   ├── main
│   │   ├── main.a
│   │   └── main.a.hash
│   └── src
│       └── microfactory.go
├── .microfactory_Linux.lock
├── .microfactory_Linux_version
├── .module_paths  `build.FindSources`的执行结果
│   ├── Android.bp.list
│   ├── Android.mk.list
│   ├── CleanSpec.mk.list
│   ├── files.db
│   └── TEST_MAPPING.list
├── ninja_build
├── .out-dir
├── soong
│   └── .soong.in_make
├── soong.log
├── soong_ui  soong_ui程序
├── .soong_ui_hash  soong_ui程序文件的hash
├── .soong_ui_intermediates  生成soong_ui程序的所用的文件、可以参考build/soong目录
│   ├── android-soong-finder
│   ├── android-soong-finder-fs
│   ├── android-soong-shared
│   ├── android-soong-ui-build
│   ├── android-soong-ui-logger
│   ├── android-soong-ui-tracer
│   ├── github.com-google-blueprint-microfactory
│   └── main
├── .soong_ui.lock
└── .soong_ui.trace  记录了soong_ui的生成过程


---

**build.Build**

build/soong/ui/build/build.go

> build生成的trace日志在out/build.trace.gz里面

> ninja的日志在out/ninja_log里面

> soong_ui的日志在soong.log里面

从build.Build函数执行中可以看到主要是有几个大的Step：

1. runMakeProductConfig

2. runSoong:生成soong的工具链

3. runKati：生成build-**.ninja,运行step2生成的ckati，搜集所有的Android.mk文件生成ninja文件，也就是前面提到的 out/build-aosp_arm.ninja

4. createCombinedBuildNinjaFile：将out/soong/build.ninja 和out/build-**.ninja 合成为combined-aosp_arm.ninja

5. runNinja：使用ninja build整个系统

``` golang

func Build(ctx Context, config Config, what int) {
    ...
    //在out目录下创建Android.mk、CleanSpec.mk、ninja_build
	SetupOutDir(ctx, config)
    //在out目录下创建CaseCheck.txt和casecheck.txt
	checkCaseSensitivity(ctx, config)

	ensureEmptyDirectoriesExist(ctx, config.TempDir())

	if what&BuildProductConfig != 0 {
		// Run make for product config
		runMakeProductConfig(ctx, config)
	}

	···

	if what&BuildSoong != 0 {
		// Run Soong
		runSoong(ctx, config)
	}

	if what&BuildKati != 0 {
		// Run ckati
		runKati(ctx, config)

		ioutil.WriteFile(config.LastKatiSuffixFile(), []byte(config.KatiSuffix()), 0777)
	} else {
		// Load last Kati Suffix if it exists
		if katiSuffix, err := ioutil.ReadFile(config.LastKatiSuffixFile()); err == nil {
			ctx.Verboseln("Loaded previous kati config:", string(katiSuffix))
			config.SetKatiSuffix(string(katiSuffix))
		}
	}

	// Write combined ninja file
	createCombinedBuildNinjaFile(ctx, config)

	if what&RunBuildTests != 0 {
		testForDanglingRules(ctx, config)
	}

	if what&BuildNinja != 0 {
		if !config.SkipMake() {
			installCleanIfNecessary(ctx, config)
		}

		// Run ninja
		runNinja(ctx, config)
	}
}
```

### runMakeProductConfig

通过dumpMakeVars来设置条件变量，同时打印到屏幕上

```go

	make_vars, err := dumpMakeVars(ctx, config, config.Arguments(), allVars, true)
	if err != nil {
		ctx.Fatalln("Error dumping make vars:", err)
	}

	// Print the banner like make does
	fmt.Fprintln(ctx.Stdout(), Banner(make_vars))

	// Populate the environment
	env := config.Environment()
	for _, name := range exportEnvVars {
		if make_vars[name] == "" {
			env.Unset(name)
		} else {
			env.Set(name, make_vars[name])
		}
	}

```
其运行结果，就是设置了如下条件变量：
```sh
============================================
PLATFORM_VERSION_CODENAME=REL
PLATFORM_VERSION=9
TARGET_PRODUCT=mini_emulator_x86_64
TARGET_BUILD_VARIANT=userdebug
TARGET_BUILD_TYPE=release
TARGET_ARCH=x86_64
TARGET_ARCH_VARIANT=x86_64
TARGET_2ND_ARCH=x86
TARGET_2ND_ARCH_VARIANT=x86_64
HOST_ARCH=x86_64
HOST_2ND_ARCH=x86
HOST_OS=linux
HOST_OS_EXTRA=Linux-4.15.0-65-generic-x86_64-Ubuntu-16.04.6-LTS
HOST_CROSS_OS=windows
HOST_CROSS_ARCH=x86
HOST_CROSS_2ND_ARCH=x86_64
HOST_BUILD_TYPE=release
BUILD_ID=PPR1.180610.011
OUT_DIR=out
============================================
set the Enviroment var: TARGET_PRODUCT
set the Enviroment var: TARGET_BUILD_VARIANT
unset the Enviroment var: TARGET_BUILD_APPS
unset the Enviroment var: CC_WRAPPER
unset the Enviroment var: CXX_WRAPPER
unset the Enviroment var: JAVAC_WRAPPER
unset the Enviroment var: CCACHE_COMPILERCHECK
unset the Enviroment var: CCACHE_SLOPPINESS
unset the Enviroment var: CCACHE_BASEDIR
unset the Enviroment var: CCACHE_CPP2

```



### runSoong：

**功能** 生成soong工具链,其所生成的soong工具链，都放在out/soong目录下
runSoong主要完成如下的工作：
1. 生成Android-mini_emulator_x86_64.mk,将所有的Android.mk汇集起来
2. 生成blueprint库
3. 生成soong工具集
4. 生成minibp
5. 将所有的Android.bp最终转义为build.ninja



在runSoong执行前，其out/soong目录的现状是：
```
android/out/soong$ ls -a
.  ..  .soong.in_make  soong.variables  .temp
```

**实现过程分析**

``` txt
android/out$ vi soong.log

2019/11/21 18:22:33.135936 build/soong/ui/build/exec.go:57: build/blueprint/bootstrap.bash [build/blueprint/bootstrap.bash -t]
2019/11/21 18:22:33.630439 build/soong/ui/build/exec.go:57: prebuilts/build-tools/linux-x86/bin/ninja [prebuilts/build-tools/linux-x86/bin/ninja -d keepdepfile -w dupbuild=err -j 8 -f out/soong/.minibootstrap/build.ninja]
2019/11/21 18:22:33.675051 build/soong/ui/build/exec.go:57: prebuilts/build-tools/linux-x86/bin/ninja [prebuilts/build-tools/linux-x86/bin/ninja -d keepdepfile -w dupbuild=err -j 8 -f out/soong/.bootstrap/build.ninja]
```

从上述日志分析，主要的工作是三个：

1. 执行`build/blueprint/bootstrap.bash`
2. 执行ninja，输入是`out/soong/.minibootstrap/build.ninja`
※ Run minibp to generate .bootstrap/build.ninja
3. 执行ninja，输入是`out/soong/.bootstrap/build.ninja`

> ※ Build any bootstrap_go_binary rules and dependencies -- usually the primary builder and any build or runtime dependencies. 

4. 使用soong_build,来生成build.ninja 

> ※ Run the primary builder to generate build.ninja

`andorid/out$ vi .ninja_log`里面详细记录了ninja的执行日志



**结果**
在runSoong执行后，其out/soong目录的结果是：
```
.
├── Android-mini_emulator_x86_64.mk
├── .blueprint.bootstrap
├── .bootstrap
│   ├── bin
│   │   ├── bpglob
│   │   ├── gotestmain
│   │   ├── gotestrunner
│   │   ├── loadplugins
│   │   ├── soong_build
│   │   └── soong_env
│   ├── blueprint
│   │   ├── pkg
│   │   └── test
│   ├── blueprint-bootstrap
│   │   └── pkg
│   ├── blueprint-bootstrap-bpdoc
│   │   └── pkg
│   ├── blueprint-deptools
│   │   └── pkg
│   ├── blueprint-parser
│   │   ├── pkg
│   │   └── test
│   ├── blueprint-pathtools
│   │   ├── pkg
│   │   └── test
│   ├── blueprint-proptools
│   │   ├── pkg
│   │   └── test
│   ├── bpglob
│   │   └── obj
│   ├── build.ninja
│   ├── build.ninja.d
│   ├── gotestmain
│   │   └── obj
│   ├── gotestrunner
│   │   └── obj
│   ├── hidl-soong-rules
│   │   └── pkg
│   ├── loadplugins
│   │   └── obj
│   ├── soong
│   │   └── pkg
│   ├── soong-android
│   │   ├── pkg
│   │   └── test
│   ├── soong-art
│   │   └── pkg
│   ├── soong-bpf
│   │   └── pkg
│   ├── soong_build
│   │   ├── gen
│   │   └── obj
│   ├── soong-cc
│   │   ├── pkg
│   │   └── test
│   ├── soong-cc-config
│   │   ├── pkg
│   │   └── test
│   ├── soong-clang
│   │   └── pkg
│   ├── soong-clang-prebuilts
│   │   └── pkg
│   ├── soong_env
│   │   └── obj
│   ├── soong-env
│   │   └── pkg
│   ├── soong-fluoride
│   │   └── pkg
│   ├── soong-genrule
│   │   └── pkg
│   ├── soong-java
│   │   ├── pkg
│   │   └── test
│   ├── soong-java-config
│   │   └── pkg
│   ├── soong-java-config-error_prone
│   │   └── pkg
│   ├── soong-llvm
│   │   └── pkg
│   ├── soong-phony
│   │   └── pkg
│   ├── soong-python
│   │   ├── pkg
│   │   └── test
│   ├── soong-shared
│   │   └── pkg
│   ├── soong-vts-spec
│   │   └── pkg
│   └── soong-wayland-protocol-codegen
│       └── pkg
├── build.ninja
├── build.ninja.d
├── make_vars-mini_emulator_x86_64.mk
├── .minibootstrap
│   ├── build.ninja
│   ├── minibp
│   ├── .minibp_hash
│   ├── .minibp_intermediates
│   │   ├── github.com-google-blueprint
│   │   ├── github.com-google-blueprint-bootstrap
│   │   ├── github.com-google-blueprint-bootstrap-bpdoc
│   │   ├── github.com-google-blueprint-deptools
│   │   ├── github.com-google-blueprint-parser
│   │   ├── github.com-google-blueprint-pathtools
│   │   ├── github.com-google-blueprint-proptools
│   │   └── main
│   └── .minibp.lock
├── .out-dir
├── soong.config
├── .soong.environment
├── .soong.in_make
├── soong.variables
└── .temp
```

### runKati

**功能** 将所有的Android.mk集合到一起，最终转义生成build_**.ninja,如out/build-mini_emulator_x86_64.ninja
其主要实现是通过调用ckati工具来实现，注意其使用的输入文件是`build/make/core/main.mk`,而几个关键的环境变量设定是

`BUILDING_WITH_NINJA=true ` 这里标记使用ninja，最终输出.ninja文件
`SOONG_ANDROID_MK=out/soong/Android-mini_emulator_x86_64.mk ` 这里使用soong输出的mk文件，具体如何使用目前没有深入研究
`SOONG_MAKEVARS_MK=out/soong/make_vars-mini_emulator_x86_64.mk` 这里使用soong输出的mk文件，具体如何使用目前没有深入研究


参考：具体的命令格式化如下
```sh
prebuilts/build-tools/linux-x86/bin/ckati 
--ninja 
--ninja_dir=out 
--ninja_suffix=-mini_emulator_x86_64 
--regen 
--ignore_optional_include=out/%.P 
--detect_android_echo 
--color_warnings 
--gen_all_targets 
--werror_find_emulator 
--kati_stats 
-f build/make/core/main.mk 
--use_find_emulator 
BUILDING_WITH_NINJA=true 
SOONG_ANDROID_MK=out/soong/Android-mini_emulator_x86_64.mk 
SOONG_MAKEVARS_MK=out/soong/make_vars-mini_emulator_x86_64.mk
```


