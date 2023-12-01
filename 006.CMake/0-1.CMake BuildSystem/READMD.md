# cmake-buildsystem
## Introduction
&nbsp;&nbsp;A CMake-based buildsystem is organized as a set of high-level logical targets. Each target corresponds to an executable or library, or is a custom target containing custom commands. Dependencies between the targets are expressed in the buildsystem to determine the build order and the rules for regeneration in response to change.（基于CMake的构建系统被组织为一组高级逻辑目标。每个目标都对应着一个可执行文件或库，或者包含自定义命令的自定义目标。目标之间的依赖关系在构建系统中表达，以确定构建顺序和响应变化的再生规则）

## Binary Targets
&nbsp;&nbsp;Executables and libraries are defined using the add_executable() and add_library() commands. The resulting binary files have appropriate PREFIX, SUFFIX and extensions for the platform targeted. Dependencies between binary targets are expressed using the target_link_libraries() command:(可执行文件和库是使用add_executable() 和 add_library()命令定义的。生成的二进制文件具有适合目标平台的前缀、后缀和拓展名。二进制目标之间的依赖关系使用target_link_libraries()命令表示:)
```cmake
  add_library(archive archive.cpp zip.cpp lzma.cpp)
  add_executable(zipapp zipapp.cpp)
  target_link_libraries(zipapp archive)
```

&nbsp;&nbsp;archive is defined as a STATIC library -- an archive containing objects compiled from archive.cpp, zip.cpp, and lzma.cpp. zipapp is defined as an executable formed by compiling and linking zipapp.cpp. When linking the zipapp executable, the archive static library is linked in.(archetype 被定义为静态库——archive包含从archive.cpp、zip.cpp和lzma.cpp编译的对象。zipapp 被定义为通过编译和链接zipapp.cpp形成的可执行文件。当链接zipapp可执行文件时，archive静态库被链接进来)

## Binary Executables(二进制可执行文件)
&nbsp;&nbsp;The add_executable() command defines an executable target:(add_executable()命令定义一个可执行文件:)
```cmake
   add_executable(mytool mytool.cpp)
```
&nbsp;&nbsp;Commands such as add_custom_command(), which generates rules to be run at build time can transparently use an EXECUTABLE target as a COMMAND executable. The buildsystem rules will ensure that the executable is built before attempting to run the command.(像add_custom_command()这样的命令生成在构建时运行的规则，可以透明地使用EXECUTABLE目标作为COMMAND可执行文件。构建系统规则将确保在尝试运行命令之前构建可执行文件)
- 类似于spring容器Bean之前的依赖关系吗?
  
## Binary Library Types
### Normal Libraries
&nbsp;&nbsp;By default, the add_library() command defines a STATIC library, unless a type is specified. A type may be specified when using the command:(默认情况下，除非指定类型，否则 add_library() 命令定义静态库。 使用命令时可以指定类型：)
```cmake
  add_library(archive SHARED archive.cpp zip.cpp lzma.cpp)
  add_library(archive STATIC archive.cpp zip.cpp lzma.cpp)
```
&nbsp;&nbsp;The BUILD_SHARED_LIBS variable may be enabled to change the behavior of add_library() to build shared libraries by default.(可以启用BUILD_SHARED_LIBS变量来更改add_library()的行为，默认情况下构建共享库。)

&nbsp;&nbsp;In the context of the buildsystem definition as a whole, it is largely irrelevant whether particular libraries are SHARED or STATIC -- the commands, dependency specifications and other APIs work similarly regardless of the library type. The MODULE library type is dissimilar in that it is generally not linked to -- it is not used in the right-hand-side of the target_link_libraries() command. It is a type which is loaded as a plugin using runtime techniques. If the library does not export any unmanaged symbols (e.g. Windows resource DLL, C++/CLI DLL), it is required that the library not be a SHARED library because CMake expects SHARED libraries to export at least one symbol.(在整个构建系统定义的上下文中，特定库是共享的还是静态的在很大程度上无关紧要——无论库类型如何，命令、依赖项规范和其他 API 的工作方式都是相似的。 MODULE 库类型的不同之处在于它通常不链接到——它不在 target_link_libraries() 命令的右侧使用。 它是一种使用运行时技术作为插件加载的类型。 如果库不导出任何非托管符号（例如 Windows 资源 DLL、C++/CLI DLL），则要求该库不是共享库，因为 CMake 期望共享库至少导出一个符号。)
```cmake
   add_library(archive MODULE 7z.cpp)
```

### Object Libraries
&nbsp;&nbsp;The OBJECT library type defines a non-archival collection of object files resulting from compiling the given source files. The object files collection may be used as source inputs to other targets by using the syntax \$<TARGET_OBJECTS:name>'. This is a generator expression that can be used to supply the OBJECT library content to other targets:('OBJECT库类型定义了由编译给定源文件产生的对象文件的非存档集合。通过使用\$<TARGET_OBJECTS:name>语法，可以将目标文件集合用作其他目标的源输入。这是一个生成器表达式，可用于向其他目标提供OBJECT库内容:')
```cmake
    add_library(archive OBJECT archive.cpp zip.cpp lzma.cpp)
    add_library(archiveExtras STATIC $<TARGET_OBJECTS:archive> extras.cpp)
    add_executable(test_exe $<TARGET_OBJECTS:archive> test.cpp)
```
&nbsp;&nbsp;The link (or archiving) step of those other targets will use the object files collection in addition to those from their own sources.(这些其他目标的链接(或存档)步骤除了使用来自它们自己源的文件之外，还将使用目标文件集合。)

&nbsp;&nbsp;Alternatively, object libraries may be linked into other targets:(或者，对象库可以链接到其他目标:)
```cmake
   add_library(archive OBJECT archive.cpp zip.cpp lzma.cpp)

   add_library(archiveExtras STATIC extras.cpp)
   target_link_libraries(archiveExtras PUBLIC archive)
   
   add_executable(test_exe test.cpp)
   target_link_libraries(test_exe archive)
```

&nbsp;&nbsp;The link (or archiving) step of those other targets will use the object files from OBJECT libraries that are directly linked. Additionally, usage requirements of the OBJECT libraries will be honored when compiling sources in those other targets. Furthermore, those usage requirements will propagate transitively to dependents of those other targets.(这些其他目标的链接(或存档)步骤将使用直接链接的object库中的对象文件。此外，在编译其他目标中的源代码时，将遵守OBJECT库的使用要求。此外，这些使用需求将传递到那些其他目标的依赖项。)

&nbsp;&nbsp;Object libraries may not be used as the TARGET in a use of the add_custom_command(TARGET) command signature. However, the list of objects can be used by add_custom_command(OUTPUT) or file(GENERATE) by using \$<TARGET_OBJECTS:objlib>.(在使用add_custom_command(TARGET)命令签名时，对象库不能用作TARGET。但是，对象列表可以由add_custom_command(OUTPUT)或file(GENERATE)使用$<TARGET_OBJECTS:objlib>。)

### Build Specification and Usage Requirements(构建规范和使用需求)
&nbsp;&nbsp;The target_include_directories(), target_compile_definitions() and target_compile_options() commands specify the build specifications and the usage requirements of binary targets. The commands populate the INCLUDE_DIRECTORIES, COMPILE_DEFINITIONS and COMPILE_OPTIONS target properties respectively, and/or the INTERFACE_INCLUDE_DIRECTORIES, INTERFACE_COMPILE_DEFINITIONS and INTERFACE_COMPILE_OPTIONS target properties.(target_include_directories()、target_compile_definitions() 和 target_compile_options() 命令指定二进制目标的构建规范和使用要求。 这些命令分别填充 INCLUDE_DIRECTORIES、COMPILE_DEFINITIONS 和 COMPILE_OPTIONS 目标属性，和/或 INTERFACE_INCLUDE_DIRECTORIES、INTERFACE_COMPILE_DEFINITIONS 和 INTERFACE_COMPILE_OPTIONS 目标属性。)

&nbsp;&nbsp;Each of the commands has a PRIVATE, PUBLIC and INTERFACE mode. The PRIVATE mode populates only the non-INTERFACE_ variant of the target property and the INTERFACE mode populates only the INTERFACE_ variants. The PUBLIC mode populates both variants of the respective target property. Each command may be invoked with multiple uses of each keyword:(每个命令都有 PRIVATE、PUBLIC 和 INTERFACE 模式。 PRIVATE 模式仅填充目标属性的非 INTERFACE_ 变体，而 INTERFACE 模式仅填充 INTERFACE_ 变体。 PUBLIC 模式填充相应目标属性的两个变体。 每个命令可以通过多次使用每个关键字来调用：)
```cmake
target_compile_definitions(archive
  PRIVATE BUILDING_WITH_LZMA
  INTERFACE USING_ARCHIVE_LIB
)
```
&nbsp;&nbsp;Note that usage requirements are not designed as a way to make downstreams use particular COMPILE_OPTIONS or COMPILE_DEFINITIONS etc for convenience only. The contents of the properties must be requirements, not merely recommendations or convenience.(请注意，使用要求并不是为了让下游使用特定的 COMPILE_OPTIONS 或 COMPILE_DEFINITIONS 等而设计的，只是为了方便。 属性的内容必须是要求，而不仅仅是建议或方便。)

### Target Properties