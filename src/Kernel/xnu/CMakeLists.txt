# XNU. This is the kernel for the system. There are a few targets we need to build
#   out of these sources.

file(MAKE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/xnu_header_install)

# RC_ENABLE_PRODUCT_INFO_FILTER

##### Makefile
# VERSDIR
# SRCROOT, OBJROOT, DSTROOT, SYMROOT
#
# RC_ProjectName::
#   Libsyscall, Libsyscall_headers_Sim, Libsyscall_driverkit (xcodebuild libsyscall)
#   libkxld (make -C libkern/kxld $@ USE_APPLE_PB_SUPPORT=all)
#   libkmod (xcodebuild libkmod (libkern/kmon))
#   xnu_tests (xnu_tests target)
#   xnu_debug, xnu_kasan
#
# install_kernels
#
# we should pass -j${RC_BUILD_JOBS}
# ... include

##### MakeInc.cmd:
# RC_XBS sets VERBOSE=YES
# QUIET != 0, unset --> VERBOSE=NO, PRINTF > /dev/null
#
# NOTE: xnu includes XCRUN as a variable. We need to be very careful...
# It additionally attempts to find an SDK by name:
# If SDKROOT, HOST_SDKROOT are passed in, SDKROOT_RESOLVED is not,
#   xcrun -sdk ${SDKROOT} -show-sdk-path is called, setting SDKROOT_RESOLVED.
# Then, SDKROOT is override to be ${SDKROOT_RESOLVED}
# The same is done for ${HOST_SDKROOT_RESOLVED}
# We should pass PLATFORM=MaxOSX in to avoid platform lookup.
# Additionally, SDKVERSION should be set to the equivalent macos version to our darwin version.
#
# We need to send in CMAKE_C_COMPILER, CMAKE_CXX_COMPILER, MIG_PATH, MIGCOM_PATH, STRIP_PATH,
#   LIPO_PATH, LIBTOOL_PATH, NM_PATH, UNIFDEF_PATH, DSYMUTIL_PATH, CTFCONVERT_PATH, CTFMERGE_PATH,
#   CTFINSERT_PATH, NMEDIT_PATH
#
# TODO: We should build by hand the tools that come with xnu.
#   Specifically, setsegname, kextsymboltool, decomment, installfile, replacecontents,
#    json_compilation_db, and possibly newvers.pl.
# Either this or we should pass HOST_CC, HOST_FLEX, HOST_BISON, HOST_GM4.
#####

##### MakeInc.def
# We will need to properly set KERNEL_CONFIGS, ARCH_CONFIGS, <ARCH>_MACHINE_CONFIGS
# KERNEL_CONFGS can include RELEASE DEVELOPMENT DEBUG PROFILE KASAN
# ARCH_CONFIGS can include X86_64 X86_64H ARM ARM64
# ARM64_MACHINE_CONFIGS can include T7000 T7001 S8000 S8001 T8010 T8011 BCM2837
# X86_64{,H}_MACHINE_CONFIGS can include NONE
#
# CURRENT_ARCH_CONFIG, CURRENT_KERNEL_CONFIG, CURRENT_MACHINE_CONFIG
# RC_NONARCH_CFLAGS, RC_NONARCH_CXXFLAGS need to be passed properly
# FIREHOSE_KERNEL_LIBRARY_PATH
#####

# DEVELOPER_EXTRAS_DIR

externalproject_add(xnu_headers.extproj
    PREFIX ${CMAKE_CURRENT_BINARY_DIR}/xnu_header_build
    SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}
    CONFIGURE_COMMAND true # disable auto CMake configure
    BUILD_COMMAND
        cmake -DCMAKE_MAKE_PROGRAM=${RC_HOST_BIN}/make
            -D XNU_SRC=${CMAKE_CURRENT_SOURCE_DIR}
            -D XNU_OBJ=<BINARY_DIR>
            -D CODESIGN_ALLOCATE_PATH=${RC_HOST_BIN}/codesign_allocate
            -D CTFCONVERT_PATH=${RC_HOST_BIN}/ctfconvert
            -D CTFMERGE_PATH=${RC_HOST_BIN}/ctfmerge
            -D CTFINSERT_PATH=${RC_HOST_BIN}/ctf_insert
            -D AVAILABILITY_PL_PATH=$<TARGET_FILE:availability.pl>
            -D MIG_PATH=${RC_HOST_BIN}/mig
            -D MIGCOM_PATH=${RC_HOST_BIN}/../libexec/migcom
            -D STRIP_PATH=${RC_HOST_BIN}/strip
            -D LIPO_PATH=${RC_HOST_BIN}/lipo
            -D NM_PATH=${RC_HOST_BIN}/nm
            -D NMEDIT_PATH=${RC_HOST_BIN}/nmedit
            -D LIBTOOL_PATH=${RC_HOST_BIN}/libtool
            -D UNIFDEF_PATH=${RC_HOST_BIN}/unifdef
            -D DSYMUTIL_PATH=${RC_HOST_BIN}/dsymutil
            -D SETSEGNAME=${RC_HOST_BIN}/setsegname
            -D KEXTSYMBOLTOOL=${RC_HOST_BIN}/kextsymboltool
            -D DECOMMENT=${RC_HOST_BIN}/decomment
            -D INSTALLFILE=${RC_HOST_BIN}/installfile
            -D REPLACECONTENTS=${RC_HOST_BIN}/replacecontents
            -D JSON_COMPILATION_DB=${RC_HOST_BIN}/json_compilation_db
            -P ${CMAKE_CURRENT_SOURCE_DIR}/preprocess_files.cmake
    COMMAND
        ${CMAKE_COMMAND} -E env RC_HOST_BIN=${RC_HOST_BIN}  ${RC_HOST_BIN}/make -C <SOURCE_DIR> installhdrs
            ARCH_CONFIGS=X86_64 BUILD_LTO=1 LOGCOLORS=y
            BUILD_WERROR=0 DO_CTFMERGE=0 KERNEL_CONFIGS=RELEASE
            SRCROOT=${CMAKE_CURRENT_SOURCE_DIR} OBJROOT=<BINARY_DIR>
            DSTROOT=${CMAKE_CURRENT_BINARY_DIR}/xnu_header_install
            VERBOSE=${CMAKE_VERBOSE_MAKEFILE} RC_ProjectName=xnu_headers
    INSTALL_COMMAND true # installed during build
    USES_TERMINAL_CONFIGURE TRUE
    USES_TERMINAL_BUILD TRUE
)

add_dependencies(xnu_headers.extproj availability.pl)

add_library(xnu_headers INTERFACE)
add_dependencies(xnu_headers xnu_headers.extproj)
target_include_directories(xnu_headers INTERFACE
    ${CMAKE_CURRENT_BINARY_DIR}/xnu_header_install/usr/include
)
add_library(xnu_private_headers INTERFACE)
add_dependencies(xnu_private_headers xnu_headers.extproj)
target_include_directories(xnu_private_headers INTERFACE
    ${CMAKE_CURRENT_BINARY_DIR}/xnu_header_install/usr/local/include
    ${CMAKE_CURRENT_BINARY_DIR}/xnu_header_install/System/Library/Frameworks/System.framework/Versions/B/PrivateHeaders
)
target_compile_options(xnu_private_headers INTERFACE
    -F${CMAKE_CURRENT_BINARY_DIR}/xnu_header_install/System/Library/Frameworks
)
add_library(xnu_kernel_headers INTERFACE)
add_dependencies(xnu_kernel_headers xnu_headers.extproj)
target_include_directories(xnu_kernel_headers INTERFACE
    ${CMAKE_CURRENT_BINARY_DIR}/xnu_header_install/System/Library/Frameworks/Kernel.framework/Versions/A/Headers
)
add_library(xnu_kernel_private_headers INTERFACE)
add_dependencies(xnu_kernel_private_headers xnu_headers.extproj)
target_include_directories(xnu_kernel_private_headers INTERFACE
    ${CMAKE_CURRENT_BINARY_DIR}/xnu_header_install/System/Library/Frameworks/Kernel.framework/Versions/A/PrivateHeaders
)



externalproject_add(xnu
    PREFIX ${CMAKE_CURRENT_BINARY_DIR}/xnu_build
    SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}
    CONFIGURE_COMMAND true # disable auto CMake configure
    BUILD_COMMAND
        cmake -DCMAKE_MAKE_PROGRAM=${RC_HOST_BIN}/make
            -D XNU_SRC=${CMAKE_CURRENT_SOURCE_DIR}
            -D XNU_OBJ=<BINARY_DIR>
            -D CODESIGN_ALLOCATE_PATH=${RC_HOST_BIN}/codesign_allocate
            -D CTFCONVERT_PATH=${RC_HOST_BIN}/ctfconvert
            -D CTFMERGE_PATH=${RC_HOST_BIN}/ctfmerge
            -D CTFINSERT_PATH=${RC_HOST_BIN}/ctf_insert
            -D AVAILABILITY_PL_PATH=$<TARGET_FILE:availability.pl>
            -D LD_PATH=${RC_HOST_BIN}/ld
            -D MIG_PATH=${RC_HOST_BIN}/mig
            -D MIGCOM_PATH=${RC_HOST_BIN}/../libexec/migcom
            -D STRIP_PATH=${RC_HOST_BIN}/strip
            -D LIPO_PATH=${RC_HOST_BIN}/lipo
            -D NM_PATH=${RC_HOST_BIN}/nm
            -D NMEDIT_PATH=${RC_HOST_BIN}/nmedit
            -D LIBTOOL_PATH=${RC_HOST_BIN}/libtool
            -D UNIFDEF_PATH=${RC_HOST_BIN}/unifdef
            -D DSYMUTIL_PATH=${RC_HOST_BIN}/dsymutil
            -D SETSEGNAME=${RC_HOST_BIN}/setsegname
            -D KEXTSYMBOLTOOL=${RC_HOST_BIN}/kextsymboltool
            -D DECOMMENT=${RC_HOST_BIN}/decomment
            -D INSTALLFILE=${RC_HOST_BIN}/installfile
            -D REPLACECONTENTS=${RC_HOST_BIN}/replacecontents
            -D JSON_COMPILATION_DB=${RC_HOST_BIN}/json_compilation_db
            -D FIREHOSE_KERNEL_LIBRARY_PATH=$<TARGET_FILE:libfirehose_kernel>
            -D FIREHOSE_KERNEL_HEADER_PATH=$<TARGET_PROPERTY:libfirehose_kernel,INTERFACE_INCLUDE_DIRECTORIES>
            -D PTHREAD_HEADER_PATH=$<TARGET_PROPERTY:pthread_common_headers,INTERFACE_INCLUDE_DIRECTORIES>
            -D CMAKE_ASM_COMPILER=${CMAKE_ASM_COMPILER}
            -D CMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}
            -D CMAKE_C_COMPILER=${CMAKE_C_COMPILER}
            -D CMAKE_C_FLAGS=${CMAKE_C_FLAGS}
            -D CMAKE_CXX_FLAGS=${CMAKE_CXX_FLAGS}
            -D RC_NONARCH_CFLAGS=${RC_NONARCH_CFLAGS}
            -D RC_NONARCH_CXXFLAGS=${RC_NONARCH_CXXFLAGS}
            -P ${CMAKE_CURRENT_SOURCE_DIR}/preprocess_files.cmake
    COMMAND
        ${CMAKE_COMMAND} -E env RC_HOST_BIN=${RC_HOST_BIN} ${RC_HOST_BIN}/make -C <SOURCE_DIR> install
            ARCH_CONFIGS=X86_64 BUILD_LTO=1 BUILD_WERROR=0 DO_CTFMERGE=0
            KERNEL_CONFIGS=RELEASE LOGCOLORS=y SRCROOT=${CMAKE_CURRENT_SOURCE_DIR}
            OBJROOT=<BINARY_DIR> SYMROOT=${CMAKE_CURRENT_BINARY_DIR}/xnu_sym
            DSTROOT=${CMAKE_CURRENT_BINARY_DIR}/xnu VERBOSE=${CMAKE_VERBOSE_MAKEFILE}
            RC_ProjectName=xnu
    INSTALL_COMMAND true # installed during build
    USES_TERMINAL_CONFIGURE TRUE
    USES_TERMINAL_BUILD TRUE
)
#add_dependencies(xnu xnu_headers libfirehose_kernel darwin_codesign_allocate ctfconvert.host ctfmerge.host migcom host_strip host_lipo host_nm unifdef host_nmedit availability.pl darwin_ld)
add_dependencies(xnu xnu_headers libfirehose_kernel availability.pl)

install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/xnu/System/Library/Frameworks/Kernel.framework/Versions/A/Resources DESTINATION System/Library/Frameworks/Kernel.framework/Versions/A COMPONENT BaseSystem)
install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/xnu/System/Library/Frameworks/Kernel.framework/Versions/A/Headers DESTINATION System/Library/Frameworks/Kernel.framework/Versions/A COMPONENT DeveloperTools)
install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/xnu/System/Library/Frameworks/Kernel.framework/Versions/A/PrivateHeaders DESTINATION System/Library/Frameworks/Kernel.framework/Versions/A COMPONENT DeveloperInternal)

# Note: This really ought to be associated with a custom build command like 'install_framework_symlinks' or something
install_symlink(System/Library/Frameworks/Kernel.framework/Versions/Current Versions/A COMPONENT BaseSystem)
install_symlink(System/Library/Frameworks/Kernel.framework/Resources Versions/Current/Resources COMPONENT BaseSystem)
install_symlink(System/Library/Frameworks/Kernel.framework/Headers Versions/Current/Headers COMPONENT DeveloperTools)
install_symlink(System/Library/Frameworks/Kernel.framework/PrivateHeaders Versions/Current/PrivateHeaders COMPONENT DeveloperInternal)

install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/xnu/System/Library/Frameworks/System.framework/Versions/B/PrivateHeaders DESTINATION System/Library/Frameworks/System.framework/Versions/B COMPONENT DeveloperInternal)

install_symlink(System/Library/Frameworks/System.framework/Versions/Current Versions/B COMPONENT BaseSystem)
install_symlink(System/Library/Frameworks/System.framework/PrivateHeaders Versions/Current/PrivateHeaders COMPONENT DeveloperInternal)

install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/xnu/System/Library/Frameworks/DriverKit.framework/Versions/A/Headers DESTINATION System/Library/Frameworks/DriverKit.framework/Versions/A COMPONENT DeveloperTools)
install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/xnu/System/Library/Frameworks/DriverKit.framework/Versions/A/PrivateHeaders DESTINATION System/Library/Frameworks/DriverKit.framework/Versions/A COMPONENT DeveloperInternal)

install_symlink(System/Library/Frameworks/DriverKit.framework/Versions/Current Versions/A COMPONENT BaseSystem)
install_symlink(System/Library/Frameworks/DriverKit.framework/Headers Versions/Current/Headers COMPONENT DeveloperTools)
install_symlink(System/Library/Frameworks/DriverKit.framework/PrivateHeaders Versions/Current/PrivateHeaders COMPONENT DeveloperInternal)

install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/xnu/System/Library/Frameworks/IOKit.framework/Versions/A/Headers DESTINATION System/Library/Frameworks/IOKit.framework/Versions/A COMPONENT DeveloperTools)
install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/xnu/System/Library/Frameworks/IOKit.framework/Versions/A/PrivateHeaders DESTINATION System/Library/Frameworks/IOKit.framework/Versions/A COMPONENT DeveloperInternal)

install_symlink(System/Library/Frameworks/IOKit.framework/Versions/Current Versions/A COMPONENT BaseSystem)
install_symlink(System/Library/Frameworks/IOKit.framework/Headers Versions/Current/Headers COMPONENT DeveloperTools)
install_symlink(System/Library/Frameworks/IOKit.framework/PrivateHeaders Versions/Current/PrivateHeaders COMPONENT DeveloperInternal)

install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/xnu/System/Library/Kernels DESTINATION System/Library COMPONENT BaseSystem)
install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/xnu/System/Library/Extensions DESTINATION System/Library COMPONENT BaseSystem)

install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/xnu/usr/include DESTINATION usr COMPONENT DeveloperTools)
install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/xnu/usr/lib DESTINATION usr COMPONENT BaseSystem)
install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/xnu/usr/libexec DESTINATION usr COMPONENT BaseSystem)
install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/xnu/usr/share/man DESTINATION usr/share COMPONENT BaseSystem)
install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/xnu/usr/share/misc DESTINATION usr/share COMPONENT BaseSystem)
install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/xnu/usr/local DESTINATION usr COMPONENT DeveloperInternal)
