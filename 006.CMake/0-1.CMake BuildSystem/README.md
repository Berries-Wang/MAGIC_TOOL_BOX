- [cmake-buildsystem](#cmake-buildsystem)
  - [Introduction](#introduction)
  - [Binary Targets](#binary-targets)
    - [Binary Executables(二进制可执行文件)](#binary-executables二进制可执行文件)
    - [Binary Library Types](#binary-library-types)
      - [Normal Libraries](#normal-libraries)
      - [Object Libraries](#object-libraries)
  - [Build Specification and Usage Requirements(构建规范和使用需求)](#build-specification-and-usage-requirements构建规范和使用需求)
    - [Target Properties](#target-properties)
    - [Transitive Usage Requirements(传递性使用要求)](#transitive-usage-requirements传递性使用要求)
    - [Compatible Interface Properties(兼容接口属性)](#compatible-interface-properties兼容接口属性)
    - [Property Origin Debugging](#property-origin-debugging)
    - [Build Specification with Generator Expressions  (使用生成器表达式构建规范)](#build-specification-with-generator-expressions--使用生成器表达式构建规范)
      - [Include Directories and Usage Requirements](#include-directories-and-usage-requirements)
    - [Link Libraries and Generator Expressions](#link-libraries-and-generator-expressions)
    - [Output Artifacts](#output-artifacts)
      - [Runtime Output Artifacts](#runtime-output-artifacts)
      - [Library Output Artifacts](#library-output-artifacts)
      - [Archive Output Artifacts](#archive-output-artifacts)
    - [Build Configurations](#build-configurations)
      - [Case Sensitivity](#case-sensitivity)
      - [Default And Custom Configurations](#default-and-custom-configurations)
    - [Pseudo Targets (伪目标)](#pseudo-targets-伪目标)
      - [Imported Targets](#imported-targets)
      - [Alias Targets](#alias-targets)
      - [Interface Libraries (接口库)](#interface-libraries-接口库)
  - [参考资料](#参考资料)

# cmake-buildsystem
## Introduction
&nbsp;&nbsp;A CMake-based buildsystem is organized as a set of high-level logical targets. Each target<sup>cmake 中 target 是核心概念?</sup> corresponds to an executable or library, or is a custom target containing custom commands. Dependencies between the targets are expressed in the buildsystem to determine the build order and the rules for regeneration in response to change.（基于CMake的构建系统被组织为一组高级逻辑目标。每个目标都对应着一个可执行文件或库，或者包含自定义命令的自定义目标。目标之间的依赖关系在构建系统中表达，以确定构建顺序和响应变化的再生规则）

## Binary Targets
&nbsp;&nbsp;Executables and libraries are defined using the add_executable() and add_library() commands. The resulting binary files have appropriate PREFIX, SUFFIX and extensions for the platform targeted. Dependencies between binary targets are expressed using the target_link_libraries() command:(可执行文件和库是使用add_executable() 和 add_library()命令定义的。生成的二进制文件具有适合目标平台的前缀、后缀和拓展名。二进制目标之间的依赖关系使用target_link_libraries()命令表示:)
```cmake
  add_library(archive archive.cpp zip.cpp lzma.cpp)
  add_executable(zipapp zipapp.cpp)
  target_link_libraries(zipapp archive)
```

&nbsp;&nbsp;archive is defined as a STATIC library -- an archive containing objects compiled from archive.cpp, zip.cpp, and lzma.cpp. zipapp is defined as an executable formed by compiling and linking zipapp.cpp. When linking the zipapp executable, the archive static library is linked in.(archetype 被定义为静态库——archive包含从archive.cpp、zip.cpp和lzma.cpp编译的对象。zipapp 被定义为通过编译和链接zipapp.cpp形成的可执行文件。当链接zipapp可执行文件时，archive静态库被链接进来)

### Binary Executables(二进制可执行文件)
&nbsp;&nbsp;The add_executable() command defines an executable target:(add_executable()命令定义一个可执行目标:)
```cmake
   add_executable(mytool mytool.cpp)
```
&nbsp;&nbsp;Commands such as add_custom_command(), which generates rules to be run at build time can transparently<sup>adv.显然地，易觉察地；明亮地</sup> use an EXECUTABLE target as a COMMAND executable. The buildsystem rules will ensure that the executable is built before attempting to run the command.(像add_custom_command()这样的命令生成在构建时运行的规则，可以透明地使用EXECUTABLE目标作为COMMAND可执行文件。构建系统规则将确保在尝试运行命令之前构建可执行文件)
  
### Binary Library Types
#### Normal Libraries
&nbsp;&nbsp;By default, the add_library() command defines a STATIC library, unless a type is specified. A type may be specified when using the command:(默认情况下，除非指定类型，否则 add_library() 命令定义静态库。 使用命令时可以指定类型：)
```cmake
  add_library(archive SHARED archive.cpp zip.cpp lzma.cpp)
  add_library(archive STATIC archive.cpp zip.cpp lzma.cpp)
```
&nbsp;&nbsp;The BUILD_SHARED_LIBS variable may be enabled to change the behavior of add_library() to build shared libraries by default.(可以启用BUILD_SHARED_LIBS变量来更改add_library()的行为，默认情况下构建共享库。)
```txt
    当我们使用CMake构建项目时，可以通过设置BUILD_SHARED_LIBS变量来决定是否构建共享库（SHARED）或静态库（STATIC）。如果BUILD_SHARED_LIBS变量的值为ON，则构建的库类型为SHARED；如果为OFF，则构建的库类型为STATIC。
    # set(BUILD_SHARED_LIBS ON) # 默认值是OFF，即构建静态库
```

&nbsp;&nbsp;In the context of the buildsystem definition as a whole, it is largely irrelevant<sup>adj.不相关的，不相干的</sup> whether particular libraries are SHARED or STATIC -- the commands, dependency specifications and other APIs work similarly regardless of the library type. The MODULE library type is dissimilar in that it is generally not linked to -- it is not used in the right-hand-side of the target_link_libraries() command. It is a type which is loaded as a plugin using runtime techniques. If the library does not export any unmanaged symbols (e.g. Windows resource DLL, C++/CLI DLL), it is required that the library not be a SHARED library because CMake expects SHARED libraries to export at least one symbol.(在整个构建系统定义的上下文中，特定库是共享的还是静态的在很大程度上无关紧要——无论库类型如何，命令、依赖项规范和其他 API 的工作方式都是相似的。 MODULE 库类型的不同之处在于它通常不链接到——它不在 target_link_libraries() 命令的右侧使用。 它是一种使用运行时技术作为插件加载的类型。 如果库不导出任何非托管符号（例如 Windows 资源 DLL、C++/CLI DLL），则要求该库不是共享库，因为 CMake 期望共享库至少导出一个符号。)
```cmake
   add_library(archive MODULE 7z.cpp)
```

#### Object Libraries
&nbsp;&nbsp;The OBJECT library type defines a non-archival collection of object files resulting from compiling the given source files. The object files collection may be used as source inputs to other targets by using the syntax \$\<TARGET_OBJECTS:name\>. This is a generator expression that can be used to supply the OBJECT library content to other targets:('OBJECT库类型定义了由编译给定源文件产生的对象文件的非存档集合。通过使用\$\<TARGET_OBJECTS:name\>语法，可以将目标文件集合用作其他目标的源输入。这是一个生成器表达式，可用于向其他目标提供OBJECT库内容:')
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

&nbsp;&nbsp;Object libraries may not be used as the TARGET in a use of the add_custom_command(TARGET) command signature. However, the list of objects can be used by add_custom_command(OUTPUT) or file(GENERATE) by using \$\<TARGET_OBJECTS:objlib\>.(在使用add_custom_command(TARGET)命令签名时，对象库不能用作TARGET。但是，对象列表可以由add_custom_command(OUTPUT)或file(GENERATE)使用\$\<TARGET_OBJECTS:objlib\>。)

## Build Specification and Usage Requirements(构建规范和使用需求)
&nbsp;&nbsp;The target_include_directories(), target_compile_definitions() and target_compile_options() commands specify the build specifications and the usage requirements of binary targets. The commands populate the INCLUDE_DIRECTORIES, COMPILE_DEFINITIONS and COMPILE_OPTIONS target properties respectively, and/or the INTERFACE_INCLUDE_DIRECTORIES, INTERFACE_COMPILE_DEFINITIONS and INTERFACE_COMPILE_OPTIONS target properties.(  target_include_directories()、target_compile_definitions() 和 target_compile_options() 命令指定二进制目标的构建规范和使用要求。 这些命令分别填充 INCLUDE_DIRECTORIES、COMPILE_DEFINITIONS 和 COMPILE_OPTIONS 目标属性，和/或 INTERFACE_INCLUDE_DIRECTORIES、INTERFACE_COMPILE_DEFINITIONS 和 INTERFACE_COMPILE_OPTIONS 目标属性。)

&nbsp;&nbsp;Each of the commands has a PRIVATE, PUBLIC and INTERFACE mode. The PRIVATE mode populates only the non-INTERFACE_ variant of the target property and the INTERFACE mode populates only the INTERFACE_ variants. The PUBLIC mode populates both variants of the respective target property. Each command may be invoked with multiple uses of each keyword:(每个命令都有 PRIVATE、PUBLIC 和 INTERFACE 模式。 PRIVATE 模式仅填充目标属性的非 INTERFACE_ 变体，而 INTERFACE 模式仅填充 INTERFACE_ 变体。 PUBLIC 模式填充相应目标属性的两个变体。 每个命令可以通过多次使用每个关键字来调用：)
```cmake
   target_compile_definitions(archive
     PRIVATE BUILDING_WITH_LZMA
     INTERFACE USING_ARCHIVE_LIB
   )
```
&nbsp;&nbsp;Note that usage requirements are not designed as a way to make downstreams use particular COMPILE_OPTIONS or COMPILE_DEFINITIONS etc for convenience only. The contents of the properties must be requirements, not merely recommendations or convenience.(请注意，使用要求并不是为了让下游使用特定的 COMPILE_OPTIONS 或 COMPILE_DEFINITIONS 等而设计的，只是为了方便。 属性的内容必须是要求，而不仅仅是建议或方便。)

### Target Properties
&nbsp;&nbsp;The contents of the INCLUDE_DIRECTORIES, COMPILE_DEFINITIONS and COMPILE_OPTIONS target properties are used appropriately<sup>adv.适当地，恰当地</sup> when compiling the source files of a binary target.(在编译二进制目标的源文件时，适当地使用INCLUDE_DIRECTORIES、COMPILE_DEFINITIONS和COMPILE_OPTIONS目标属性的内容。)

&nbsp;&nbsp;Entries in the INCLUDE_DIRECTORIES are added to the compile line with -I or -isystem prefixes and in the order of appearance in the property value.(INCLUDE_DIRECTORIES中的条目以-I或- issystem前缀添加到编译行，并按照属性值中出现的顺序添加。)

&nbsp;&nbsp;Entries in the COMPILE_DEFINITIONS are prefixed with -D or /D and added to the compile line in an unspecified order. The DEFINE_SYMBOL target property is also added as a compile definition as a special convenience case for SHARED and MODULE library targets.(COMPILE_DEFINITIONS中的项以-D或/D为前缀，并以未指定的顺序添加到编译行中。DEFINE_SYMBOL target属性也作为编译定义添加，作为SHARED和MODULE库目标的特殊方便情况。)
```cmake
   # 如文档：https://cmake.org/cmake/help/latest/prop_tgt/COMPILE_DEFINITIONS.html，在cmake3.26版本，-D前导被移除了。

   # 在新版本中，可以使用'add_compile_definitions()'来进行宏定义，不用加-D前缀
   # 为了引入'stm32f10x_conf.h' 头文件，见: stm32f10x.h
   add_compile_definitions(USE_STDPERIPH_DRIVER)

```

&nbsp;&nbsp;Entries in the COMPILE_OPTIONS are escaped for the shell and added in the order of appearance in the property value. Several compile options have special separate handling, such as POSITION_INDEPENDENT_CODE.（COMPILE_OPTIONS中的项将为shell进行转义，并按照出现的顺序添加到属性值中。一些编译选项有特殊的单独处理，比如POSITION_INDEPENDENT_CODE。）

&nbsp;&nbsp;The contents of the INTERFACE_INCLUDE_DIRECTORIES, INTERFACE_COMPILE_DEFINITIONS and INTERFACE_COMPILE_OPTIONS target properties are Usage Requirements -- they specify content which consumers must use to correctly compile and link with the target they appear on. For any binary target, the contents of each INTERFACE_ property on each target specified in a target_link_libraries() command is consumed:（INTERFACE_INCLUDE_DIRECTORIES、INTERFACE_COMPILE_DEFINITIONS和INTERFACE_COMPILE_OPTIONS目标属性的内容是Usage Requirements——它们指定了消费者必须使用的内容，以便正确编译并链接到它们出现的目标。对于任何二进制目标，在target_link_libraries()命令中指定的每个目标上的每个INTERFACE_属性的内容被消耗:）
```cmake
    set(srcs archive.cpp zip.cpp)
    if (LZMA_FOUND)
      list(APPEND srcs lzma.cpp)
    endif()
    add_library(archive SHARED ${srcs})
    if (LZMA_FOUND)
      # The archive library sources are compiled with -DBUILDING_WITH_LZMA
      target_compile_definitions(archive PRIVATE BUILDING_WITH_LZMA)
    endif()
    target_compile_definitions(archive INTERFACE USING_ARCHIVE_LIB)
    
    add_executable(consumer)
    # Link consumer to archive and consume its usage requirements. The consumer
    # executable sources are compiled with -DUSING_ARCHIVE_LIB.
    target_link_libraries(consumer archive)
```
&nbsp;&nbsp;Because it is common to require that the source directory and corresponding build directory are added to the INCLUDE_DIRECTORIES, the CMAKE_INCLUDE_CURRENT_DIR variable can be enabled to conveniently add the corresponding directories to the INCLUDE_DIRECTORIES of all targets. The variable CMAKE_INCLUDE_CURRENT_DIR_IN_INTERFACE can be enabled to add the corresponding directories to the INTERFACE_INCLUDE_DIRECTORIES of all targets. This makes use of targets in multiple different directories convenient through use of the target_link_libraries() command.(因为通常需要将源目录和相应的构建目录添加到INCLUDE_DIRECTORIES中，所以可以启用CMAKE_INCLUDE_CURRENT_DIR变量，方便地将相应的目录添加到所有目标的INCLUDE_DIRECTORIES中。CMAKE_INCLUDE_CURRENT_DIR_IN_INTERFACE变量可以在所有目标器的INTERFACE_INCLUDE_DIRECTORIES中添加相应的目录。通过使用target_link_libraries()命令，可以方便地使用多个不同目录中的目标。)

### Transitive Usage Requirements(传递性使用要求)
&nbsp;&nbsp;The usage requirements of a target can transitively propagate to the dependents. The target_link_libraries() command has PRIVATE, INTERFACE and PUBLIC keywords to control the propagation. (目标的使用需求可以传递地传播给依赖对象。target_link_libraries()命令有PRIVATE、INTERFACE和PUBLIC关键字来控制传播。)
```cmake
   add_library(archive archive.cpp)
   target_compile_definitions(archive INTERFACE USING_ARCHIVE_LIB)
   
   add_library(serialization serialization.cpp)
   target_compile_definitions(serialization INTERFACE USING_SERIALIZATION_LIB)
   
   add_library(archiveExtras extras.cpp)
   target_link_libraries(archiveExtras PUBLIC archive)
   target_link_libraries(archiveExtras PRIVATE serialization)
   # archiveExtras is compiled with -DUSING_ARCHIVE_LIB
   # and -DUSING_SERIALIZATION_LIB
   
   add_executable(consumer consumer.cpp)
   # consumer is compiled with -DUSING_ARCHIVE_LIB
   target_link_libraries(consumer archiveExtras)
```
&nbsp;&nbsp;Because the archive is a PUBLIC dependency of archiveExtras, the usage requirements of it are propagated to consumer too.(因为archive是archiveExtras的PUBLIC依赖项，因此，他的使用需求也会传播给consumer)

&nbsp;&nbsp;Because serialization is a PRIVATE dependency of archiveExtras, the usage requirements of it are not propagated to consumer.(因为serialization是archiveExtras的PRIVATE依赖项，所以它的使用需求不会传播给consumer。)

&nbsp;&nbsp;Generally, a dependency should be specified in a use of target_link_libraries() with the PRIVATE keyword if it is used by only the implementation of a library, and not in the header files. If a dependency is additionally used in the header files of a library (e.g. for class inheritance), then it should be specified as a PUBLIC dependency. A dependency which is not used by the implementation of a library, but only by its headers should be specified as an INTERFACE dependency. The target_link_libraries() command may be invoked with multiple uses of each keyword:(通常，如果依赖项仅由库的实现使用，而不是在头文件中使用，则应该在使用target_link_libraries()时使用PRIVATE关键字指定它。如果依赖项在库的头文件中被额外使用(例如用于类继承)，那么它应该被指定为PUBLIC依赖项。不被库的实现使用而只被其头文件使用的依赖项应该指定为INTERFACE依赖项。target_link_libraries()命令可以通过多次使用每个关键字来调用:)
```cmake
  target_link_libraries(archiveExtras
    PUBLIC archive
    PRIVATE serialization
  )
```

&nbsp;&nbsp;Usage requirements are propagated by reading the INTERFACE_ variants of target properties from dependencies and appending the values to the non-INTERFACE_ variants of the operand. For example, the INTERFACE_INCLUDE_DIRECTORIES of dependencies is read and appended to the INCLUDE_DIRECTORIES of the operand. In cases where order is relevant and maintained, and the order resulting from the target_link_libraries() calls does not allow correct compilation, use of an appropriate command to set the property directly may update the order.(使用需求通过从依赖项中读取目标属性的INTERFACE_变量并将值附加到操作数的非INTERFACE_变量来传播。例如，读取依赖项的INTERFACE_INCLUDE_DIRECTORIES并将其附加到操作数的INCLUDE_DIRECTORIES中。如果顺序是相关的和维护的，并且target_link_libraries()调用产生的顺序不允许正确编译，则使用适当的命令直接设置属性可能会更新顺序。)

&nbsp;&nbsp;For example, if the linked libraries for a target must be specified in the order lib1 lib2 lib3 , but the include directories must be specified in the order lib3 lib1 lib2:(例如，如果目标的链接库必须按顺序lib1 lib2 lib3指定，但包含目录必须按顺序lib3 lib1 lib2指定:)
```cmake
    target_link_libraries(myExe lib1 lib2 lib3)
    target_include_directories(myExe
      PRIVATE $<TARGET_PROPERTY:lib3,INTERFACE_INCLUDE_DIRECTORIES>)
```
&nbsp;&nbsp;Note that care must be taken when specifying usage requirements for targets which will be exported for installation using the install(EXPORT) command(注意，在指定将使用install(EXPORT)命令导出用于安装的目标的使用需求时，必须非常小心)

### Compatible Interface Properties(兼容接口属性)
&nbsp;&nbsp;Some target properties are required to be compatible between a target and the interface of each dependency. For example, the POSITION_INDEPENDENT_CODE target property may specify a boolean value of whether a target should be compiled as position-independent-code, which has platform-specific consequences. A target may also specify the usage requirement INTERFACE_POSITION_INDEPENDENT_CODE to communicate that consumers must be compiled as position-independent-code.(某些目标属性需要在目标和每个依赖项的接口之间兼容。例如，POSITION_INDEPENDENT_CODE target属性可以指定是否应该将目标编译为位置无关代码的布尔值，这具有特定于平台的结果。目标还可以指定使用需求INTERFACE_POSITION_INDEPENDENT_CODE，以传达消费者必须被编译为与位置无关的代码。)
```cmake
   add_executable(exe1 exe1.cpp)
   set_property(TARGET exe1 PROPERTY POSITION_INDEPENDENT_CODE ON)
   
   add_library(lib1 SHARED lib1.cpp)
   set_property(TARGET lib1 PROPERTY INTERFACE_POSITION_INDEPENDENT_CODE ON)
   
   add_executable(exe2 exe2.cpp)
   target_link_libraries(exe2 lib1)
```
&nbsp;&nbsp;Here, both exe1 and exe2 will be compiled as position-independent-code.  lib1 will also be compiled as position-independent-code because that is the default setting for SHARED libraries.f dependencies have conflicting, non-compatible requirements cmake(1) issues a diagnostic:(在这里，exe1和exe2都将被编译为与位置无关的代码。lib1也将被编译为与位置无关的代码，因为这是共享库的默认设置。如果依赖项有冲突的、不兼容的需求，make(1)会发出诊断:)
```cmake
    add_library(lib1 SHARED lib1.cpp)
    set_property(TARGET lib1 PROPERTY INTERFACE_POSITION_INDEPENDENT_CODE ON)
    
    add_library(lib2 SHARED lib2.cpp)
    set_property(TARGET lib2 PROPERTY INTERFACE_POSITION_INDEPENDENT_CODE OFF)
    
    add_executable(exe1 exe1.cpp)
    target_link_libraries(exe1 lib1)
    set_property(TARGET exe1 PROPERTY POSITION_INDEPENDENT_CODE OFF)
    
    add_executable(exe2 exe2.cpp)
    target_link_libraries(exe2 lib1 lib2)
```

&nbsp;&nbsp;The lib1 requirement INTERFACE_POSITION_INDEPENDENT_CODE is not "compatible" with the POSITION_INDEPENDENT_CODE property of the exe1 target. The library requires that consumers are built as position-independent-code, while the executable specifies to not built as position-independent-code, so a diagnostic is issued.(lib1要求INTERFACE_POSITION_INDEPENDENT_CODE与ex1目标的POSITION_INDEPENDENT_CODE属性不“兼容”。库要求将消费者构建为与位置无关的代码，而可执行文件指定不构建为与位置无关的代码，因此会发出诊断。)

&nbsp;&nbsp;The lib1 and lib2 requirements are not "compatible". One of them requires that consumers are built as position-independent-code, while the other requires that consumers are not built as position-independent-code. Because exe2 links to both and they are in conflict, a CMake error message is issued:(lib1和lib2的要求不“兼容”。其中一个要求将消费者构建为与位置无关的代码，而另一个要求不将消费者构建为与位置无关的代码。因为ex2链接到两者并且它们是冲突的，所以会发出一个CMake错误消息:)
```txt
   CMake Error: The INTERFACE_POSITION_INDEPENDENT_CODE property of "lib2" does not agree with the value of POSITION_INDEPENDENT_CODE already determined for "exe2".
```

&nbsp;&nbsp;To be "compatible", the POSITION_INDEPENDENT_CODE property, if set must be either the same, in a boolean sense, as the INTERFACE_POSITION_INDEPENDENT_CODE property of all transitively specified dependencies on which that property is set.(为了“兼容”，POSITION_INDEPENDENT_CODE属性，如果设置，必须在布尔意义上与设置该属性的所有传递指定依赖项的INTERFACE_POSITION_INDEPENDENT_CODE属性相同。)

&nbsp;&nbsp;This property of "compatible interface requirement" may be extended to other properties by specifying the property in the content of the COMPATIBLE_INTERFACE_BOOL target property. Each specified property must be compatible between the consuming target and the corresponding property with an INTERFACE_ prefix from each dependency:(他的“兼容接口要求”属性可以通过在COMPATIBLE_INTERFACE_BOOL目标属性的内容中指定该属性来扩展到其他属性。每个指定的属性必须在消费目标和对应的属性之间兼容，这些属性带有每个依赖项的INTERFACE_前缀:)
```cmake
  add_library(lib1Version2 SHARED lib1_v2.cpp)
  set_property(TARGET lib1Version2 PROPERTY INTERFACE_CUSTOM_PROP ON)
  set_property(TARGET lib1Version2 APPEND PROPERTY
    COMPATIBLE_INTERFACE_BOOL CUSTOM_PROP
  )
  
  add_library(lib1Version3 SHARED lib1_v3.cpp)
  set_property(TARGET lib1Version3 PROPERTY INTERFACE_CUSTOM_PROP OFF)
  
  add_executable(exe1 exe1.cpp)
  target_link_libraries(exe1 lib1Version2) # CUSTOM_PROP will be ON
  
  add_executable(exe2 exe2.cpp)
  target_link_libraries(exe2 lib1Version2 lib1Version3) # Diagnostic
```

&nbsp;&nbsp;Non-boolean properties may also participate in "compatible interface" computations. Properties specified in the COMPATIBLE_INTERFACE_STRING property must be either unspecified or compare to the same string among all transitively specified dependencies. This can be useful to ensure that multiple incompatible versions of a library are not linked together through transitive requirements of a target:(非布尔属性也可能参与“兼容接口”计算。在COMPATIBLE_INTERFACE_STRING属性中指定的属性必须未指定，或者在所有传递指定的依赖项中与相同的字符串进行比较。这对于确保库的多个不兼容版本不会通过目标的传递需求链接在一起非常有用:)
```cmake
    add_library(lib1Version2 SHARED lib1_v2.cpp)
    set_property(TARGET lib1Version2 PROPERTY INTERFACE_LIB_VERSION 2)
    set_property(TARGET lib1Version2 APPEND PROPERTY
      COMPATIBLE_INTERFACE_STRING LIB_VERSION
    )
    
    add_library(lib1Version3 SHARED lib1_v3.cpp)
    set_property(TARGET lib1Version3 PROPERTY INTERFACE_LIB_VERSION 3)
    
    add_executable(exe1 exe1.cpp)
    target_link_libraries(exe1 lib1Version2) # LIB_VERSION will be "2"
    
    add_executable(exe2 exe2.cpp)
    target_link_libraries(exe2 lib1Version2 lib1Version3) # Diagnostic
```

&nbsp;&nbsp;The COMPATIBLE_INTERFACE_NUMBER_MAX target property specifies that content will be evaluated numerically and the maximum number among all specified will be calculated:(COMPATIBLE_INTERFACE_NUMBER_MAX目标属性指定将对内容进行数值计算，并计算所有指定的最大值:)
```cmake
    add_library(lib1Version2 SHARED lib1_v2.cpp)
    set_property(TARGET lib1Version2 PROPERTY INTERFACE_CONTAINER_SIZE_REQUIRED 200)
    set_property(TARGET lib1Version2 APPEND PROPERTY
      COMPATIBLE_INTERFACE_NUMBER_MAX CONTAINER_SIZE_REQUIRED
    )
    
    add_library(lib1Version3 SHARED lib1_v3.cpp)
    set_property(TARGET lib1Version3 PROPERTY INTERFACE_CONTAINER_SIZE_REQUIRED 1000)
    
    add_executable(exe1 exe1.cpp)
    # CONTAINER_SIZE_REQUIRED will be "200"
    target_link_libraries(exe1 lib1Version2)
    
    add_executable(exe2 exe2.cpp)
    # CONTAINER_SIZE_REQUIRED will be "1000"
    target_link_libraries(exe2 lib1Version2 lib1Version3)
```

&nbsp;&nbsp;Similarly, the COMPATIBLE_INTERFACE_NUMBER_MIN may be used to calculate the numeric minimum value for a property from dependencies.(类似地，COMPATIBLE_INTERFACE_NUMBER_MIN可以用于从依赖项中计算属性的数值最小值。)

&nbsp;&nbsp;Each calculated "compatible" property value may be read in the consumer at generate-time using generator expressions.(每个计算出来的“compatible”属性值都可以在生成时使用生成器表达式在消费者中读取。)

&nbsp;&nbsp;Note that for each dependee, the set of properties specified in each compatible interface property must not intersect with the set specified in any of the other properties.(请注意，对于每个依赖项，每个兼容接口属性中指定的属性集不得与任何其他属性中指定的属性集相交。)

### Property Origin Debugging
&nbsp;&nbsp;Because build specifications can be determined by dependencies, the lack of locality of code which creates a target and code which is responsible for setting build specifications may make the code more difficult to reason about. [cmake(1)](https://cmake.org/cmake/help/latest/manual/cmake.1.html#manual:cmake(1)) provides a debugging facility to print the origin of the contents of properties which may be determined by dependencies. The properties which can be debugged are listed in the CMAKE_DEBUG_TARGET_PROPERTIES variable documentation:(由于构建规范可以通过依赖关系来确定，因此创建目标的代码和负责设置构建规范的代码缺乏局部性可能会使代码更难以推理。 [cmake(1)](https://cmake.org/cmake/help/latest/manual/cmake.1.html#manual:cmake(1)) 提供了一个调试工具来打印可能由依赖项确定的属性内容的来源。 可以调试的属性在 CMAKE_DEBUG_TARGET_PROPERTIES 变量文档中列出：)
```cmake
   set(CMAKE_DEBUG_TARGET_PROPERTIES
     INCLUDE_DIRECTORIES
     COMPILE_DEFINITIONS
     POSITION_INDEPENDENT_CODE
     CONTAINER_SIZE_REQUIRED
     LIB_VERSION
   )
   add_executable(exe1 exe1.cpp)
```

&nbsp;&nbsp;In the case of properties listed in COMPATIBLE_INTERFACE_BOOL or COMPATIBLE_INTERFACE_STRING, the debug output shows which target was responsible for setting the property, and which other dependencies also defined the property. In the case of COMPATIBLE_INTERFACE_NUMBER_MAX and COMPATIBLE_INTERFACE_NUMBER_MIN, the debug output shows the value of the property from each dependency, and whether the value determines the new extreme.(对于 COMPATIBLE_INTERFACE_BOOL 或 COMPATIBLE_INTERFACE_STRING 中列出的属性，调试输出显示哪个目标负责设置该属性，以及哪些其他依赖项也定义了该属性。 对于 COMPATIBLE_INTERFACE_NUMBER_MAX 和 COMPATIBLE_INTERFACE_NUMBER_MIN，调试输出显示每个依赖项的属性值，以及该值是否确定新的极值。)

### Build Specification with Generator Expressions  (使用生成器表达式构建规范)
&nbsp;&nbsp;Build specifications may use generator expressions containing content which may be conditional or known only at generate-time. For example, the calculated "compatible" value of a property may be read with the TARGET_PROPERTY expression:(构建规范可以使用包含可能是有条件的或仅在生成时已知的内容的生成器表达式。 例如，可以使用 TARGET_PROPERTY 表达式读取计算出的属性的“兼容”值：)
```cmake
   add_library(lib1Version2 SHARED lib1_v2.cpp)
   set_property(TARGET lib1Version2 PROPERTY
     INTERFACE_CONTAINER_SIZE_REQUIRED 200)
   set_property(TARGET lib1Version2 APPEND PROPERTY
     COMPATIBLE_INTERFACE_NUMBER_MAX CONTAINER_SIZE_REQUIRED
   )
   
   add_executable(exe1 exe1.cpp)
   target_link_libraries(exe1 lib1Version2)
   target_compile_definitions(exe1 PRIVATE
       CONTAINER_SIZE=$<TARGET_PROPERTY:CONTAINER_SIZE_REQUIRED>
   )
```

&nbsp;&nbsp;In this case, the exe1 source files will be compiled with -DCONTAINER_SIZE=200.

&nbsp;&nbsp;The unary TARGET_PROPERTY generator expression and the TARGET_POLICY generator expression are evaluated with the consuming target context. This means that a usage requirement specification may be evaluated differently based on the consumer:(一元TARGET_PROPERTY生成器表达式和TARGET_POLICY生成器表达式使用消费目标上下文进行计算。这意味着使用需求规范可能会根据消费者的不同进行不同的评估:)
```cmake
    add_library(lib1 lib1.cpp)
    target_compile_definitions(lib1 INTERFACE
      $<$<STREQUAL:$<TARGET_PROPERTY:TYPE>,EXECUTABLE>:LIB1_WITH_EXE>
      $<$<STREQUAL:$<TARGET_PROPERTY:TYPE>,SHARED_LIBRARY>:LIB1_WITH_SHARED_LIB>
      $<$<TARGET_POLICY:CMP0041>:CONSUMER_CMP0041_NEW>
    )
    
    add_executable(exe1 exe1.cpp)
    target_link_libraries(exe1 lib1)
    
    cmake_policy(SET CMP0041 NEW)
    
    add_library(shared_lib shared_lib.cpp)
    target_link_libraries(shared_lib lib1)
```

&nbsp;&nbsp;The exe1 executable will be compiled with -DLIB1_WITH_EXE, while the shared_lib shared library will be compiled with -DLIB1_WITH_SHARED_LIB and -DCONSUMER_CMP0041_NEW, because policy CMP0041 is NEW at the point where the shared_lib target is created.(exe1可执行文件将使用-DLIB1_WITH_EXE编译，而shared_lib共享库将使用-DLIB1_WITH_SHARED_LIB和-DCONSUMER_CMP0041_NEW编译，因为在创建shared_lib目标时，策略CMP0041是NEW。)

&nbsp;&nbsp;The BUILD_INTERFACE expression wraps requirements which are only used when consumed from a target in the same buildsystem, or when consumed from a target exported to the build directory using the export() command. The INSTALL_INTERFACE expression wraps requirements which are only used when consumed from a target which has been installed and exported with the install(EXPORT) command:(BUILD_INTERFACE 表达式包装了仅在从同一构建系统中的目标使用时使用的需求，或者从使用 export() 命令导出到构建目录的目标使用时使用的需求。 INSTALL_INTERFACE 表达式包含仅在从已使用 install(EXPORT) 命令安装和导出的目标使用时使用的要求)
```cmake
   add_library(ClimbingStats climbingstats.cpp)
   target_compile_definitions(ClimbingStats INTERFACE
     $<BUILD_INTERFACE:ClimbingStats_FROM_BUILD_LOCATION>
     $<INSTALL_INTERFACE:ClimbingStats_FROM_INSTALLED_LOCATION>
   )
   install(TARGETS ClimbingStats EXPORT libExport ${InstallArgs})
   install(EXPORT libExport NAMESPACE Upstream::
           DESTINATION lib/cmake/ClimbingStats)
   export(EXPORT libExport NAMESPACE Upstream::)
   
   add_executable(exe1 exe1.cpp)
   target_link_libraries(exe1 ClimbingStats)
```

&nbsp;&nbsp;In this case, the exe1 executable will be compiled with -DClimbingStats_FROM_BUILD_LOCATION. The exporting commands generate IMPORTED targets with either the INSTALL_INTERFACE or the BUILD_INTERFACE omitted, and the *_INTERFACE marker stripped away. A separate project consuming the ClimbingStats package would contain:(在这种情况下，exe1 可执行文件将使用 -DClimbingStats_FROM_BUILD_LOCATION 进行编译。 导出命令生成 IMPORTED 目标，其中省略了 INSTALL_INTERFACE 或 BUILD_INTERFACE，并且删除了 *_INTERFACE 标记。 使用 ClimbingStats 包的单独项目将包含)
```cmake
   find_package(ClimbingStats REQUIRED)
   
   add_executable(Downstream main.cpp)
   target_link_libraries(Downstream Upstream::ClimbingStats)
```

&nbsp;&nbsp;Depending on whether the ClimbingStats package was used from the build location or the install location, the Downstream target would be compiled with either -DClimbingStats_FROM_BUILD_LOCATION or -DClimbingStats_FROM_INSTALL_LOCATION. For more about packages and exporting see the cmake-packages(7) manual.(根据 ClimbingStats 包是从构建位置还是安装位置使用，下游目标将使用 -DClimbingStats_FROM_BUILD_LOCATION 或 -DClimbingStats_FROM_INSTALL_LOCATION 进行编译。 有关包和导出的更多信息，请参阅 cmake-packages(7) 手册。)

#### Include Directories and Usage Requirements
&nbsp;&nbsp;Include directories require some special consideration when specified as usage requirements and when used with generator expressions. The target_include_directories() command accepts both relative and absolute include directories:(当指定为使用要求以及与生成器表达式一起使用时，包含目录需要一些特殊考虑。 target_include_directories() 命令接受相对和绝对包含目录：)
```cmake
   add_library(lib1 lib1.cpp)
   target_include_directories(lib1 PRIVATE
     /absolute/path
     relative/path
   )
```

&nbsp;&nbsp;Relative paths are interpreted relative to the source directory where the command appears. Relative paths are not allowed in the INTERFACE_INCLUDE_DIRECTORIES of IMPORTED targets.(相对路径是相对于命令出现的源目录进行解释的。 IMPORTED 目标的 INTERFACE_INCLUDE_DIRECTORIES 中不允许使用相对路径。)

&nbsp;&nbsp;In cases where a non-trivial generator expression is used, the INSTALL_PREFIX expression may be used within the argument of an INSTALL_INTERFACE expression. It is a replacement marker which expands to the installation prefix when imported by a consuming project.(在使用重要生成器表达式的情况下，可以在INSTALL_INTERFACE表达式的参数中使用INSTALL_PREFIX表达式。它是一个替换标记，在被消费项目导入时扩展为安装前缀。)

&nbsp;&nbsp;Include directories usage requirements commonly differ between the build-tree and the install-tree. The BUILD_INTERFACE and INSTALL_INTERFACE generator expressions can be used to describe separate usage requirements based on the usage location. Relative paths are allowed within the INSTALL_INTERFACE expression and are interpreted relative to the installation prefix. For example:(包含目录的使用需求通常在构建树和安装树之间有所不同。BUILD_INTERFACE和INSTALL_INTERFACE生成器表达式可用于描述基于使用位置的单独使用需求。在INSTALL_INTERFACE表达式中允许使用相对路径，并且相对于安装前缀进行解释。例如:)
```cmake
    add_library(ClimbingStats climbingstats.cpp)
    target_include_directories(ClimbingStats INTERFACE
      $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/generated>
      $<INSTALL_INTERFACE:/absolute/path>
      $<INSTALL_INTERFACE:relative/path>
      $<INSTALL_INTERFACE:$<INSTALL_PREFIX>/$<CONFIG>/generated>
    )
```
&nbsp;&nbsp;Two convenience APIs are provided relating to include directories usage requirements. The CMAKE_INCLUDE_CURRENT_DIR_IN_INTERFACE variable may be enabled, with an equivalent effect to:(提供了两个与包含目录使用需求相关的便利api。可以启用CMAKE_INCLUDE_CURRENT_DIR_IN_INTERFACE变量，其效果与)
```cmake
   set_property(TARGET tgt APPEND PROPERTY INTERFACE_INCLUDE_DIRECTORIES
     $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR};${CMAKE_CURRENT_BINARY_DIR}>
   )
```

&nbsp;&nbsp;for each target affected. The convenience for installed targets is an INCLUDES DESTINATION component with the install(TARGETS) command:(对于每个受影响的目标。安装目标的方便之处是使用install(targets)命令的INCLUDES DESTINATION组件:)
```cmake
   install(TARGETS foo bar bat EXPORT tgts ${dest_args}
     INCLUDES DESTINATION include
   )
   install(EXPORT tgts ${other_args})
   install(FILES ${headers} DESTINATION include)
```

&nbsp;&nbsp;This is equivalent to appending \${CMAKE_INSTALL_PREFIX}/include to the INTERFACE_INCLUDE_DIRECTORIES of each of the installed IMPORTED targets when generated by install(EXPORT).(这相当于在install(EXPORT)生成时，将\${CMAKE_INSTALL_PREFIX}/include附加到每个已安装的导入目标的INTERFACE_INCLUDE_DIRECTORIES。)

&nbsp;&nbsp;When the INTERFACE_INCLUDE_DIRECTORIES of an imported target is consumed, the entries in the property may be treated as system include directories. The effects of that are toolchain-dependent, but one common effect is to omit compiler warnings for headers found in those directories. The SYSTEM property of the installed target determines this behavior (see the EXPORT_NO_SYSTEM property for how to modify the installed value for a target). It is also possible to change how consumers interpret the system behavior of consumed imported targets by setting the NO_SYSTEM_FROM_IMPORTED target property on the consumer.(当使用导入目标的INTERFACE_INCLUDE_DIRECTORIES时，属性中的条目可能被视为系统包含目录。这样做的效果依赖于工具链，但一个常见的效果是忽略在这些目录中找到的头文件的编译器警告。已安装目标的SYSTEM属性决定了这种行为(有关如何修改目标的已安装值，请参阅EXPORT_NO_SYSTEM属性)。还可以通过在消费者上设置NO_SYSTEM_FROM_IMPORTED target属性来更改消费者解释已使用导入目标的系统行为的方式。)

&nbsp;&nbsp;If a binary target is linked transitively to a macOS FRAMEWORK, the Headers directory of the framework is also treated as a usage requirement. This has the same effect as passing the framework directory as an include directory.(如果二进制目标被传递地链接到macOS框架，框架的Headers目录也被视为使用需求。这与将框架目录作为包含目录传递具有相同的效果。)

### Link Libraries and Generator Expressions
&nbsp;&nbsp;Like build specifications, link libraries may be specified with generator expression conditions. However, as consumption of usage requirements is based on collection from linked dependencies, there is an additional limitation that the link dependencies must form a "directed acyclic graph". That is, if linking to a target is dependent on the value of a target property, that target property may not be dependent on the linked dependencies:(与构建规范一样，可以使用生成器表达式条件指定链接库。然而，由于使用需求的消费是基于链接依赖项的收集，因此存在一个额外的限制，即链接依赖项必须形成“有向无环图”。也就是说，如果链接到目标依赖于目标属性的值，那么该目标属性可能不依赖于链接的依赖项)
```cmake
   add_library(lib1 lib1.cpp)
   add_library(lib2 lib2.cpp)
   target_link_libraries(lib1 PUBLIC
     $<$<TARGET_PROPERTY:POSITION_INDEPENDENT_CODE>:lib2>
   )
   add_library(lib3 lib3.cpp)
   set_property(TARGET lib3 PROPERTY INTERFACE_POSITION_INDEPENDENT_CODE ON)
   
   add_executable(exe1 exe1.cpp)
   target_link_libraries(exe1 lib1 lib3)
```

&nbsp;&nbsp;As the value of the POSITION_INDEPENDENT_CODE property of the exe1 target is dependent on the linked libraries (lib3), and the edge of linking exe1 is determined by the same POSITION_INDEPENDENT_CODE property, the dependency graph above contains a cycle. cmake(1) issues an error message.(由于exe1目标的POSITION_INDEPENDENT_CODE属性的值依赖于链接的库(lib3)，并且链接exe1的边缘由相同的POSITION_INDEPENDENT_CODE属性确定，因此上面的依赖关系图包含一个循环。Cmake(1)发出错误消息。)

### Output Artifacts
&nbsp;&nbsp;The buildsystem targets created by the add_library() and add_executable() commands create rules to create binary outputs. The exact output location of the binaries can only be determined at generate-time because it can depend on the build-configuration and the link-language of linked dependencies etc. TARGET_FILE, TARGET_LINKER_FILE and related expressions can be used to access the name and location of generated binaries. These expressions do not work for OBJECT libraries however, as there is no single file generated by such libraries which is relevant to the expressions.(由add_library()和add_executable()命令创建的buildsystem目标创建规则来创建二进制输出。二进制文件的确切输出位置只能在生成时确定，因为它可能取决于构建配置和链接依赖项的链接语言等。TARGET_FILE、TARGET_LINKER_FILE和相关表达式可用于访问生成的二进制文件的名称和位置。然而，这些表达式不适用于OBJECT库，因为这些库没有生成与表达式相关的单个文件。)

&nbsp;&nbsp;There are three kinds of output artifacts that may be build by targets as detailed in the following sections. Their classification differs between DLL platforms and non-DLL platforms. All Windows-based systems including Cygwin are DLL platforms.(目标可以构建三种类型的输出工件，如下面的部分所述。它们的分类在DLL平台和非DLL平台之间有所不同。包括Cygwin在内的所有基于windows的系统都是DLL平台。)

#### Runtime Output Artifacts
&nbsp;&nbsp;A runtime output artifact of a buildsystem target may be:
+ The executable file (e.g. .exe) of an executable target created by the add_executable() command.
+ On DLL platforms: the executable file (e.g. .dll) of a shared library target created by the add_library() command with the SHARED option.

&nbsp;&nbsp;The RUNTIME_OUTPUT_DIRECTORY and RUNTIME_OUTPUT_NAME target properties may be used to control runtime output artifact locations and names in the build tree.(RUNTIME_OUTPUT_DIRECTORY和RUNTIME_OUTPUT_NAME目标属性可用于控制构建树中的运行时输出工件位置和名称。)

#### Library Output Artifacts
&nbsp;&nbsp;A library output artifact of a buildsystem target may be:
+ The loadable module file (e.g. .dll or .so) of a module library target created by the add_library() command with the MODULE option.
+ On non-DLL platforms: the shared library file (e.g. .so or .dylib) of a shared library target created by the add_library() command with the SHARED option.

&nbsp;&nbsp;The LIBRARY_OUTPUT_DIRECTORY and LIBRARY_OUTPUT_NAME target properties may be used to control library output artifact locations and names in the build tree.(LIBRARY_OUTPUT_DIRECTORY和LIBRARY_OUTPUT_NAME目标属性可用于控制构建树中的库输出工件位置和名称。)

#### Archive Output Artifacts
&nbsp;&nbsp;An archive output artifact of a buildsystem target may be:
+ The static library file (e.g. .lib or .a) of a static library target created by the add_library() command with the STATIC option.
  
+ On DLL platforms: the import library file (e.g. .lib) of a shared library target created by the add_library() command with the SHARED option. This file is only guaranteed to exist if the library exports at least one unmanaged symbol.
  
+ On DLL platforms: the import library file (e.g. .lib) of an executable target created by the add_executable() command when its ENABLE_EXPORTS target property is set.
  
+ On AIX: the linker import file (e.g. .imp) of an executable target created by the add_executable() command when its ENABLE_EXPORTS target property is set.
  
+ On macOS: the linker import file (e.g. .tbd) of a shared library target created by the add_library() command with the SHARED option and when its ENABLE_EXPORTS target property is set.

&nbsp;&nbsp;The ARCHIVE_OUTPUT_DIRECTORY and ARCHIVE_OUTPUT_NAME target properties may be used to control archive output artifact locations and names in the build tree.(ARCHIVE_OUTPUT_DIRECTORY和ARCHIVE_OUTPUT_NAME目标属性可用于控制构建树中的存档输出工件位置和名称。)

### Build Configurations
&nbsp;&nbsp;Configurations determine specifications for a certain type of build, such as Release or Debug. The way this is specified depends on the type of generator being used. For single configuration generators like Makefile Generators and Ninja, the configuration is specified at configure time by the CMAKE_BUILD_TYPE variable. For multi-configuration generators like Visual Studio, Xcode, and Ninja Multi-Config, the configuration is chosen by the user at build time and CMAKE_BUILD_TYPE is ignored. In the multi-configuration case, the set of available configurations is specified at configure time by the CMAKE_CONFIGURATION_TYPES variable, but the actual configuration used cannot be known until the build stage. This difference is often misunderstood, leading to problematic code like the following:(配置确定特定类型的构建的规范，例如发布或调试。指定它的方式取决于所使用的生成器的类型。对于单个配置生成器，如Makefile generators和Ninja，配置是在配置时由CMAKE_BUILD_TYPE变量指定的。对于像Visual Studio, Xcode和Ninja Multi-Config这样的多配置生成器，配置由用户在构建时选择，CMAKE_BUILD_TYPE被忽略。在多配置的情况下，可用配置集在配置时由CMAKE_CONFIGURATION_TYPES变量指定，但是直到构建阶段才能知道所使用的实际配置。这种区别经常被误解，导致出现如下问题代码:)
```cmake
   # WARNING: This is wrong for multi-config generators because they don't use
   #          and typically don't even set CMAKE_BUILD_TYPE
   string(TOLOWER ${CMAKE_BUILD_TYPE} build_type)
   if (build_type STREQUAL debug)
     target_compile_definitions(exe1 PRIVATE DEBUG_BUILD)
   endif()
```

&nbsp;&nbsp;Generator expressions should be used instead to handle configuration-specific logic correctly, regardless of the generator used. For example:(Generator expressions should be used instead to handle configuration-specific logic correctly, regardless of the generator used. For example)(应该使用生成器表达式来正确处理特定于配置的逻辑，而不管使用的是哪种生成器。例如:应该使用生成器表达式来正确处理特定于配置的逻辑，而不管使用的是哪种生成器。例如))
```cmake
   # Works correctly for both single and multi-config generators
   target_compile_definitions(exe1 PRIVATE
     $<$<CONFIG:Debug>:DEBUG_BUILD>
   )
```

&nbsp;&nbsp;In the presence of IMPORTED targets, the content of MAP_IMPORTED_CONFIG_DEBUG is also accounted for by the above \$\<CONFIG:Debug\> expression.(在导入目标存在的情况下，map_importted_config_debug的内容也由上面的$\<CONFIG:Debug>表达式负责。)

#### Case Sensitivity
&nbsp;&nbsp;CMAKE_BUILD_TYPE and CMAKE_CONFIGURATION_TYPES are just like other variables in that any string comparisons made with their values will be case-sensitive. The \$\<CONFIG\> generator expression also preserves the casing of the configuration as set by the user or CMake defaults. For example: (CMAKE_BUILD_TYPE和CMAKE_CONFIGURATION_TYPES就像其他变量一样，任何与它们的值进行的字符串比较都是区分大小写的。$\<CONFIG\>生成器表达式还保留由用户或CMake默认设置的配置的大小写。例如:)
```cmake
    # NOTE: Don't use these patterns, they are for illustration purposes only.
    
    set(CMAKE_BUILD_TYPE Debug)
    if(CMAKE_BUILD_TYPE STREQUAL DEBUG)
      # ... will never get here, "Debug" != "DEBUG"
    endif()
    add_custom_target(print_config ALL
      # Prints "Config is Debug" in this single-config case
      COMMAND ${CMAKE_COMMAND} -E echo "Config is $<CONFIG>"
      VERBATIM
    )
    
    set(CMAKE_CONFIGURATION_TYPES Debug Release)
    if(DEBUG IN_LIST CMAKE_CONFIGURATION_TYPES)
      # ... will never get here, "Debug" != "DEBUG"
    endif()
```

&nbsp;&nbsp;In contrast, CMake treats the configuration type case-insensitively when using it internally in places that modify behavior based on the configuration. For example, the \$\<CONFIG:Debug\> generator expression will evaluate to 1 for a configuration of not only Debug, but also DEBUG, debug or even DeBuG. Therefore, you can specify configuration types in CMAKE_BUILD_TYPE and CMAKE_CONFIGURATION_TYPES with any mixture of upper and lowercase, although there are strong conventions (see the next section). If you must test the value in string comparisons, always convert the value to upper or lowercase first and adjust the test accordingly.(相比之下，CMake在内部根据配置修改行为的地方使用配置类型时，不区分大小写。例如，\$\<CONFIG:Debug\>生成器表达式对于不仅为Debug的配置，而且为Debug、DEBUG甚至debug的配置的值为1。因此，您可以在CMAKE_BUILD_TYPE和CMAKE_CONFIGURATION_TYPES中指定任意大小写混合的配置类型，尽管有很强的约定(参见下一节)。如果必须在字符串比较中测试值，请始终先将值转换为大写或小写，然后相应地调整测试。)

#### Default And Custom Configurations
By default, CMake defines a number of standard configurations:(默认情况下，CMake定义了一些标准配置:)
+ Debug
+ Release
+ RelWithDebInfo
+ MinSizeRel

&nbsp;&nbsp;In multi-config generators, the CMAKE_CONFIGURATION_TYPES variable will be populated with (potentially a subset of) the above list by default, unless overridden by the project or user. The actual configuration used is selected by the user at build time.(在多配置生成器中，CMAKE_CONFIGURATION_TYPES变量将默认使用(可能是上述列表的一个子集)填充，除非被项目或用户覆盖。所使用的实际配置由用户在构建时选择。)

&nbsp;&nbsp;For single-config generators, the configuration is specified with the CMAKE_BUILD_TYPE variable at configure time and cannot be changed at build time. The default value will often be none of the above standard configurations and will instead be an empty string. A common misunderstanding is that this is the same as Debug, but that is not the case. Users should always explicitly specify the build type instead to avoid this common problem.(对于单配置生成器，配置是在配置时用CMAKE_BUILD_TYPE变量指定的，不能在构建时更改。默认值通常不是上述标准配置中的任何一个，而是一个空字符串。一个常见的误解是，这与Debug相同，但事实并非如此。用户应该始终显式地指定构建类型，以避免这个常见问题。)

&nbsp;&nbsp;The above standard configuration types provide reasonable behavior on most platforms, but they can be extended to provide other types. Each configuration defines a set of compiler and linker flag variables for the language in use. These variables follow the convention CMAKE_\<LANG>\_FLAGS\_\<CONFIG\>, where \<CONFIG\> is always the uppercase configuration name. When defining a custom configuration type, make sure these variables are set appropriately, typically as cache variables.(上述标准配置类型在大多数平台上提供了合理的行为，但它们可以扩展以提供其他类型。每个配置为所使用的语言定义了一组编译器和链接器标志变量。这些变量遵循约定CMAKE_\<LANG>\_FLAGS\_\<CONFIG\>，其中\<CONFIG\>始终是大写配置名。在定义自定义配置类型时，请确保适当地设置这些变量，通常设置为缓存变量)

### Pseudo Targets (伪目标)
&nbsp;&nbsp;Some target types do not represent outputs of the buildsystem, but only inputs such as external dependencies, aliases or other non-build artifacts. Pseudo targets are not represented in the generated buildsystem.(一些目标类型并不表示构建系统的输出，而只是外部依赖、别名或其他非构建工件等输入。生成的构建系统中没有表示伪目标。)

#### Imported Targets
&nbsp;&nbsp;An IMPORTED target represents a pre-existing dependency. Usually such targets are defined by an upstream package and should be treated as immutable. After declaring an IMPORTED target one can adjust its target properties by using the customary commands such as target_compile_definitions(), target_include_directories(), target_compile_options() or target_link_libraries() just like with any other regular target.(导入的目标表示预先存在的依赖项。通常这样的目标是由上游包定义的，应该被视为不可变的。在声明了导入的目标之后，就可以像处理其他常规目标一样，使用target_compile_definitions()、target_include_directories()、target_compile_options()或target_link_libraries()等常规命令来调整其目标属性。)

&nbsp;&nbsp;IMPORTED targets may have the same usage requirement properties populated as binary targets, such as INTERFACE_INCLUDE_DIRECTORIES, INTERFACE_COMPILE_DEFINITIONS, INTERFACE_COMPILE_OPTIONS, INTERFACE_LINK_LIBRARIES, and INTERFACE_POSITION_INDEPENDENT_CODE.

&nbsp;&nbsp;The LOCATION may also be read from an IMPORTED target, though there is rarely reason to do so. Commands such as add_custom_command() can transparently use an IMPORTED EXECUTABLE target as a COMMAND executable.(导入的目标可能具有与二进制目标相同的使用需求属性，例如INTERFACE_INCLUDE_DIRECTORIES、INTERFACE_COMPILE_DEFINITIONS、INTERFACE_COMPILE_OPTIONS、INTERFACE_LINK_LIBRARIES和INTERFACE_POSITION_INDEPENDENT_CODE。)

&nbsp;&nbsp;The scope of the definition of an IMPORTED target is the directory where it was defined. It may be accessed and used from subdirectories, but not from parent directories or sibling directories. The scope is similar to the scope of a cmake variable.(已导入目标的定义范围是定义该目标的目录。它可以从子目录访问和使用，但不能从父目录或兄弟目录访问和使用。作用域类似于cmake变量的作用域。)

&nbsp;&nbsp;It is also possible to define a GLOBAL IMPORTED target which is accessible globally in the buildsystem.(也可以定义一个全局导入的目标，它可以在构建系统中全局访问。)

&nbsp;&nbsp;See the cmake-packages(7) manual for more on creating packages with IMPORTED targets.

#### Alias Targets
&nbsp;&nbsp;An ALIAS target is a name which may be used interchangeably with a binary target name in read-only contexts. A primary use-case for ALIAS targets is for example or unit test executables accompanying a library, which may be part of the same buildsystem or built separately based on user configuration.(ALIAS目标是一个可以在只读上下文中与二进制目标名称互换使用的名称。ALIAS目标的主要用例是伴随库的单元测试可执行文件，它可能是相同构建系统的一部分，也可能是基于用户配置单独构建的。)
```cmake
   add_library(lib1 lib1.cpp)
   install(TARGETS lib1 EXPORT lib1Export ${dest_args})
   install(EXPORT lib1Export NAMESPACE Upstream:: ${other_args})
   
   add_library(Upstream::lib1 ALIAS lib1)
```

&nbsp;&nbsp;In another directory, we can link unconditionally to the Upstream::lib1 target, which may be an IMPORTED target from a package, or an ALIAS target if built as part of the (在另一个目录中，我们可以无条件地链接到Upstream::lib1目标，它可以是从包中导入的目标，也可以是作为同一构建系统的一部分构建的ALIAS目标。)same buildsystem.
```cmake
  if (NOT TARGET Upstream::lib1)
    find_package(lib1 REQUIRED)
  endif()
  add_executable(exe1 exe1.cpp)
  target_link_libraries(exe1 Upstream::lib1)
```

&nbsp;&nbsp;ALIAS targets are not mutable, installable or exportable. They are entirely local to the buildsystem description. A name can be tested for whether it is an ALIAS name by reading the ALIASED_TARGET property from it:(ALIAS目标不可改变、不可安装或不可导出。它们完全是构建系统描述的局部内容。可以通过读取ALIASED_TARGET属性来测试一个名称是否为别名:)
```cmake
  get_target_property(_aliased Upstream::lib1 ALIASED_TARGET)
  if(_aliased)
    message(STATUS "The name Upstream::lib1 is an ALIAS for ${_aliased}.")
  endif()
```

#### Interface Libraries (接口库)
&nbsp;&nbsp;An INTERFACE library target does not compile sources and does not produce a library artifact on disk, so it has no LOCATION.(INTERFACE库目标不编译源代码，也不会在磁盘上生成库工件，因此它没有LOCATION。)

&nbsp;&nbsp;It may specify usage requirements such as INTERFACE_INCLUDE_DIRECTORIES, INTERFACE_COMPILE_DEFINITIONS, INTERFACE_COMPILE_OPTIONS, INTERFACE_LINK_LIBRARIES, INTERFACE_SOURCES, and INTERFACE_POSITION_INDEPENDENT_CODE. Only the INTERFACE modes of the target_include_directories(), target_compile_definitions(), target_compile_options(), target_sources(), and target_link_libraries() commands may be used with INTERFACE libraries.(它可以指定使用需求，如INTERFACE_INCLUDE_DIRECTORIES, INTERFACE_COMPILE_DEFINITIONS, INTERFACE_COMPILE_OPTIONS, INTERFACE_LINK_LIBRARIES, INTERFACE_SOURCES，和INTERFACE_POSITION_INDEPENDENT_CODE。只有target_include_directories()、target_compile_definitions()、target_compile_options()、target_sources()和target_link_libraries()命令的INTERFACE模式可以与INTERFACE库一起使用。)

&nbsp;&nbsp;Since CMake 3.19, an INTERFACE library target may optionally contain source files. An interface library that contains source files will be included as a build target in the generated buildsystem. It does not compile sources, but may contain custom commands to generate other sources. Additionally, IDEs will show the source files as part of the target for interactive reading and editing.(从CMake 3.19开始，INTERFACE库目标可以选择性地包含源文件。包含源文件的接口库将作为生成的构建系统中的构建目标。它不编译源代码，但可能包含用于生成其他源代码的自定义命令。此外，ide将源文件显示为目标文件的一部分，以便进行交互式阅读和编辑。)

&nbsp;&nbsp;A primary use-case for INTERFACE libraries is header-only libraries. Since CMake 3.23, header files may be associated with a library by adding them to a header set using the target_sources() command:(INTERFACE库的主要用例是仅头文件库。从CMake 3.23开始，头文件可以通过使用target_sources()命令将它们添加到头文件集来与库关联:)
```cmake
   add_library(Eigen INTERFACE)
   
   target_sources(Eigen PUBLIC
     FILE_SET HEADERS
       BASE_DIRS src
       FILES src/eigen.h src/vector.h src/matrix.h
   )
   
   add_executable(exe1 exe1.cpp)
   target_link_libraries(exe1 Eigen)
```

&nbsp;&nbsp;When we specify the FILE_SET here, the BASE_DIRS we define automatically become include directories in the usage requirements for the target Eigen. The usage requirements from the target are consumed and used when compiling, but have no effect on linking.(当我们在这里指定FILE_SET时，我们定义的BASE_DIRS自动成为目标特征的使用需求中的包含目录。来自目标的使用需求在编译时被消耗和使用，但对链接没有影响。)

&nbsp;&nbsp;Another use-case is to employ an entirely target-focussed design for usage requirements:(另一个用例是为使用需求采用完全以目标为中心的设计:)
```cmake
    add_library(pic_on INTERFACE)
    set_property(TARGET pic_on PROPERTY INTERFACE_POSITION_INDEPENDENT_CODE ON)
    add_library(pic_off INTERFACE)
    set_property(TARGET pic_off PROPERTY INTERFACE_POSITION_INDEPENDENT_CODE OFF)
    
    add_library(enable_rtti INTERFACE)
    target_compile_options(enable_rtti INTERFACE
      $<$<OR:$<COMPILER_ID:GNU>,$<COMPILER_ID:Clang>>:-rtti>
    )
    
    add_executable(exe1 exe1.cpp)
    target_link_libraries(exe1 pic_on enable_rtti)
```

&nbsp;&nbsp;This way, the build specification of exe1 is expressed entirely as linked targets, and the complexity of compiler-specific flags is encapsulated in an INTERFACE library target.(这样，exe1的构建规范完全表示为链接的目标，并且特定于编译器标志的复杂性被封装在一个INTERFACE库目标中。)

&nbsp;&nbsp;INTERFACE libraries may be installed and exported. We can install the default header set along with the target:(可以安装和导出INTERFACE库。我们可以将默认头文件集与目标文件一起安装:)
```cmake
   add_library(Eigen INTERFACE)
   
   target_sources(Eigen INTERFACE
     FILE_SET HEADERS
       BASE_DIRS src
       FILES src/eigen.h src/vector.h src/matrix.h
   )
   
   install(TARGETS Eigen EXPORT eigenExport
     FILE_SET HEADERS DESTINATION include/Eigen)
   install(EXPORT eigenExport NAMESPACE Upstream::
     DESTINATION lib/cmake/Eigen
   )
```

&nbsp;&nbsp;Here, the headers defined in the header set are installed to include/Eigen. The install destination automatically becomes an include directory that is a usage requirement for consumers.(在这里，头集合中定义的头被安装为包含/Eigen。安装目标自动成为一个包含目录，这是消费者的使用需求。)

## 参考资料
1. [cmake-buildsystem](https://cmake.org/cmake/help/latest/manual/cmake-buildsystem.7.html#pseudo-targets)




