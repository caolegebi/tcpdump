cmake_minimum_required(VERSION 2.8.6)

set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake/Modules)

project(tcpdump)

#
# Try to enable as many C99 features as we can.
# At minimum, we want C++/C99-style // comments.
#
# Newer versions of compilers might default to supporting C99, but older
# versions may require a special flag.
#
# Prior to CMake 3.1, setting CMAKE_C_STANDARD will not have any effect,
# so, unless and until we require CMake 3.1 or later, we have to do it
# ourselves on pre-3.1 CMake, so we just do it ourselves on all versions
# of CMake.
#
# Note: with CMake 3.1 through 3.5, the only compilers for which CMake
# handles CMAKE_C_STANDARD are GCC and Clang.  3.6 adds support only
# for Intel C; 3.9 adds support for PGI C, Sun C, and IBM XL C, and
# 3.10 adds support for Cray C and IAR C, but no version of CMake has
# support for HP C.  Therefore, even if we use CMAKE_C_STANDARD with
# compilers for which CMake supports it, we may still have to do it
# ourselves on other compilers.
#
# See the CMake documentation for the CMAKE_<LANG>_COMPILER_ID variables
# for a list of compiler IDs.
#
# We don't worry about MSVC; it doesn't have such a flag - either it
# doesn't support the C99 features we need at all, or it supports them
# regardless of the compiler flag.
#
# XXX - this just tests whether the option works and adds it if it does.
# We don't test whether it's necessary in order to get the C99 features
# that we use; if we ever have a user who tries to compile with a compiler
# that can't be made to support those features, we can add a test to make
# sure we actually *have* C99 support.
#
include(CheckCCompilerFlag)
macro(check_and_add_compiler_option _option)
    message(STATUS "Checking C compiler flag ${_option}")
    string(REPLACE "=" "-" _temp_option_variable ${_option})
    string(REGEX REPLACE "^-" "" _option_variable ${_temp_option_variable})
    check_c_compiler_flag("${_option}" ${_option_variable})
    if(${${_option_variable}})
        set(C_ADDITIONAL_FLAGS "${C_ADDITIONAL_FLAGS} ${_option}")
    endif()
endmacro()

set(C_ADDITIONAL_FLAGS "")
if(CMAKE_C_COMPILER_ID MATCHES "GNU" OR
   CMAKE_C_COMPILER_ID MATCHES "Clang")
    check_and_add_compiler_option("-std=gnu99")
elseif(CMAKE_C_COMPILER_ID MATCHES "XL")
    #
    # We want support for extensions picked up for GNU C compatibility,
    # so we use -qlanglvl=extc99.
    #
    check_and_add_compiler_option("-qlanglvl=extc99")
elseif(CMAKE_C_COMPILER_ID MATCHES "HP")
    check_and_add_compiler_option("-AC99")
elseif(CMAKE_C_COMPILER_ID MATCHES "Sun")
    check_and_add_compiler_option("-xc99")
elseif(CMAKE_C_COMPILER_ID MATCHES "Intel")
    check_and_add_compiler_option("-c99")
endif()

set(LIBRARY_NAME netdissect)

###################################################################
#   Parameters
###################################################################

option(WITH_SMI "Build with libsmi, if available" ON)
option(WITH_CRYPTO "Build with OpenSSL/libressl libcrypto, if available" ON)
option(WITH_CAPSICUM "Build with Capsicum security functions, if available" ON)
option(WITH_CAP_NG "Use libcap-ng, if available" ON)
option(ENABLE_SMB "Build with the SMB dissector" ON)

#
# String parameters.  Neither of them are set, initially; only if the
# user explicitly configures them are they set.
#
# WITH_CHROOT is STRING, not PATH, as the directory need not exist
# when CMake is run.
#
set(WITH_CHROOT CACHE STRING
    "Directory to which to chroot when dropping privileges")
set(WITH_USER CACHE STRING
    "User to whom to set the UID when dropping privileges")

#
# By default, build universal with the appropriate set of architectures
# for the OS on which we're doing the build.
#
if(APPLE AND "${CMAKE_OSX_ARCHITECTURES}" STREQUAL "")
    #
    # Get the major version of Darwin.
    #
    string(REGEX MATCH "^([0-9]+)" SYSTEM_VERSION_MAJOR "${CMAKE_SYSTEM_VERSION}")

    if(SYSTEM_VERSION_MAJOR EQUAL 9)
        #
        # Leopard.  Build for x86 and 32-bit PowerPC, with
        # x86 first.  (That's what Apple does.)
        #
        set(CMAKE_OSX_ARCHITECTURES "i386;ppc")
    elseif(SYSTEM_VERSION_MAJOR EQUAL 10)
        #
        # Snow Leopard.  Build for x86-64 and x86, with
        # x86-64 first.  (That's what Apple does.)
        #
        set(CMAKE_OSX_ARCHITECTURES "x86_64;i386")
    endif()
endif()

###################################################################
#   Versioning
###################################################################

# Get, parse, format and set tcpdump's version string from
# [tcpdump_root]/VERSION for later use.

# Get MAJOR, MINOR, PATCH & SUFFIX
file(STRINGS ${tcpdump_SOURCE_DIR}/VERSION
    PACKAGE_VERSION
    LIMIT_COUNT 1 # Read only the first line
)

######################################
# Project settings
######################################

add_definitions(-DHAVE_CONFIG_H)

include_directories(
    ${CMAKE_CURRENT_BINARY_DIR}
    ${tcpdump_SOURCE_DIR}
)

if(MSVC)
    add_definitions(-D__STDC__)
    add_definitions(-D_CRT_SECURE_NO_WARNINGS)
endif(MSVC)

if(MSVC)
    if (USE_STATIC_RT)
        MESSAGE(STATUS "Use STATIC runtime")
        set(NAME_RT MT)
        set (CMAKE_CXX_FLAGS_MINSIZEREL     "${CMAKE_CXX_FLAGS_MINSIZEREL} /MT")
        set (CMAKE_CXX_FLAGS_RELWITHDEBINFO "${CMAKE_CXX_FLAGS_RELWITHDEBINFO} /MT")
        set (CMAKE_CXX_FLAGS_RELEASE        "${CMAKE_CXX_FLAGS_RELEASE} /MT")
        set (CMAKE_CXX_FLAGS_DEBUG          "${CMAKE_CXX_FLAGS_DEBUG} /MTd")

        set (CMAKE_C_FLAGS_MINSIZEREL       "${CMAKE_C_FLAGS_MINSIZEREL} /MT")
        set (CMAKE_C_FLAGS_RELWITHDEBINFO   "${CMAKE_C_FLAGS_RELWITHDEBINFO} /MT")
        set (CMAKE_C_FLAGS_RELEASE          "${CMAKE_C_FLAGS_RELEASE} /MT")
        set (CMAKE_C_FLAGS_DEBUG            "${CMAKE_C_FLAGS_DEBUG} /MTd")
    else (USE_STATIC_RT)
        MESSAGE(STATUS "Use DYNAMIC runtime")
        set(NAME_RT MD)
        set (CMAKE_CXX_FLAGS_MINSIZEREL     "${CMAKE_CXX_FLAGS_MINSIZEREL} /MD")
        set (CMAKE_CXX_FLAGS_RELWITHDEBINFO "${CMAKE_CXX_FLAGS_RELWITHDEBINFO} /MD")
        set (CMAKE_CXX_FLAGS_RELEASE        "${CMAKE_CXX_FLAGS_RELEASE} /MD")
        set (CMAKE_CXX_FLAGS_DEBUG          "${CMAKE_CXX_FLAGS_DEBUG} /MDd")

        set (CMAKE_C_FLAGS_MINSIZEREL       "${CMAKE_C_FLAGS_MINSIZEREL} /MD")
        set (CMAKE_C_FLAGS_RELWITHDEBINFO   "${CMAKE_C_FLAGS_RELWITHDEBINFO} /MD")
        set (CMAKE_C_FLAGS_RELEASE          "${CMAKE_C_FLAGS_RELEASE} /MD")
        set (CMAKE_C_FLAGS_DEBUG            "${CMAKE_C_FLAGS_DEBUG} /MDd")
   endif (USE_STATIC_RT)
endif(MSVC)

###################################################################
#   Detect available platform features
###################################################################

include(CMakePushCheckState)
include(CheckIncludeFile)
include(CheckIncludeFiles)
include(CheckFunctionExists)
include(CheckLibraryExists)
include(CheckSymbolExists)
include(CheckStructHasMember)
include(CheckVariableExists)
include(CheckTypeSize)

#
# Header files.
#
check_include_file(fcntl.h HAVE_FCNTL_H)
check_include_file(rpc/rpc.h HAVE_RPC_RPC_H)
check_include_file(net/if.h HAVE_NET_IF_H)
if(HAVE_RPC_RPC_H)
    check_include_files("rpc/rpc.h;rpc/rpcent.h" HAVE_RPC_RPCENT_H)
endif(HAVE_RPC_RPC_H)
if(NOT WIN32)
    check_include_files("sys/types.h;sys/socket.h;net/if.h;net/pfvar.h" HAVE_NET_PFVAR_H)
    if(HAVE_NET_PFVAR_H)
        check_include_files("sys/types.h;sys/socket.h;net/if.h;net/pfvar.h;net/if_pflog.h" HAVE_NET_IF_PFLOG_H)
        if(HAVE_NET_IF_PFLOG_H)
            set(LOCALSRC print-pflog.c ${LOCALSRC})
        endif(HAVE_NET_IF_PFLOG_H)
    endif(HAVE_NET_PFVAR_H)
endif(NOT WIN32)

#
# Functions.
#
check_function_exists(strlcat HAVE_STRLCAT)
check_function_exists(strlcpy HAVE_STRLCPY)
check_function_exists(strdup HAVE_STRDUP)
check_function_exists(strsep HAVE_STRSEP)

#
# Find library needed for gethostbyaddr.
# NOTE: if you hand check_library_exists as its last argument a variable
# that's been set, it skips the test, so we need different variables.
#
set(TCPDUMP_LINK_LIBRARIES "")
if(WIN32)
    #
    # We need winsock2.h and ws2tcpip.h.
    #
    cmake_push_check_state()
    set(CMAKE_REQUIRED_LIBRARIES ws2_32)
    check_symbol_exists(gethostbyaddr "winsock2.h;ws2tcpip.h" LIBWS2_32_HAS_GETHOSTBYADDR)
    cmake_pop_check_state()
    if(LIBWS2_32_HAS_GETHOSTBYADDR)
        set(TCPDUMP_LINK_LIBRARIES ws2_32 ${TCPDUMP_LINK_LIBRARIES})
    else(LIBWS2_32_HAS_GETHOSTBYADDR)
        message(FATAL_ERROR "gethostbyaddr is required, but wasn't found")
    endif(LIBWS2_32_HAS_GETHOSTBYADDR)
else(WIN32)
    check_function_exists(gethostbyaddr STDLIBS_HAVE_GETHOSTBYADDR)
    if(NOT STDLIBS_HAVE_GETHOSTBYADDR)
        check_library_exists(socket gethostbyaddr "" LIBSOCKET_HAS_GETHOSTBYADDR)
        if(LIBSOCKET_HAS_GETHOSTBYADDR)
            set(TCPDUMP_LINK_LIBRARIES ${TCPDUMP_LINK_LIBRARIES} socket)
        else(LIBSOCKET_HAS_GETHOSTBYADDR)
            check_library_exists(nsl gethostbyaddr "" LIBNSL_HAS_GETHOSTBYADDR)
            if(LIBNSL_HAS_GETHOSTBYADDR)
                set(TCPDUMP_LINK_LIBRARIES ${TCPDUMP_LINK_LIBRARIES} nsl)
            else(LIBNSL_HAS_GETHOSTBYADDR)
                message(FATAL_ERROR "gethostbyaddr is required, but wasn't found")
            endif(LIBNSL_HAS_GETHOSTBYADDR)
        endif(LIBSOCKET_HAS_GETHOSTBYADDR)
    endif(NOT STDLIBS_HAVE_GETHOSTBYADDR)
endif(WIN32)

#
# This may require additional libraries.
#
cmake_push_check_state()
set(CMAKE_REQUIRED_LIBRARIES ${TCPDUMP_LINK_LIBRARIES})
check_function_exists(getservent STDLIBS_HAVE_GETSERVENT)
if(STDLIBS_HAVE_GETSERVENT)
    set(HAVE_GETSERVENT TRUE)
else(STDLIBS_HAVE_GETSERVENT)
    #
    # Some platforms may need -lsocket for getservent.
    #
    set(CMAKE_REQUIRED_LIBRARIES socket ${TCPDUMP_LINK_LIBRARIES})
    check_function_exists(getservent LIBSOCKET_HAS_GETSERVENT)
    if(LIBSOCKET_HAS_GETSERVENT)
        set(HAVE_GETSERVENT TRUE)
        set(TCPDUMP_LINK_LIBRARIES socket ${TCPDUMP_LINK_LIBRARIES})
    endif(LIBSOCKET_HAS_GETSERVENT)
endif(STDLIBS_HAVE_GETSERVENT)
cmake_pop_check_state()

check_function_exists(getopt_long HAVE_GETOPT_LONG)
#
# For Windows, either
#
#   1) we're using VS 2015, in which case we have both snprintf()
#      and vsnprintf(), and they behave in a C99-compliant fashion,
#      so we use them
#
# or
#
#   2) we're not, and we don't have snprintf(), and we either don't
#      have vsnprintf() or we have one that *doesn't* behave in a
#      C99-compliant fashion, but we *do* have _snprintf_s() and
#      _vsnprintf_s(), so we wrap them with #defines
#
# and we test for both of them at compile time, so we don't need to
# check for snprintf() or vsnprintf() here.
#
# XXX - do we need to care about UN*Xes that don't have snprintf()
# or vsnprintf() any more?
#
# We also don't need to waste time checking for fork() or vfork().
#
if(NOT WIN32)
  check_function_exists(vsnprintf HAVE_VSNPRINTF)
  check_function_exists(snprintf HAVE_SNPRINTF)
  check_function_exists(fork HAVE_FORK)
  check_function_exists(vfork HAVE_VFORK)
endif(NOT WIN32)
check_function_exists(strftime HAVE_STRFTIME)
check_function_exists(setlinebuf HAVE_SETLINEBUF)

#
# Some platforms may need -lnsl for getrpcbynumber.
#
cmake_push_check_state()
set(CMAKE_REQUIRED_LIBRARIES ${TCPDUMP_LINK_LIBRARIES})
check_function_exists(getrpcbynumber STDLIBS_HAVE_GETRPCBYNUMBER)
if(STDLIBS_HAVE_GETRPCBYNUMBER)
    set(HAVE_GETRPCBYNUMBER TRUE)
else(STDLIBS_HAVE_GETRPCBYNUMBER)
    set(CMAKE_REQUIRED_LIBRARIES ${TCPDUMP_LINK_LIBRARIES} nsl)
    check_function_exists(getrpcbynumber LIBNSL_HAS_GETRPCBYNUMBER)
    if(LIBNSL_HAS_GETRPCBYNUMBER)
        set(HAVE_GETRPCBYNUMBER TRUE)
        set(TCPDUMP_LINK_LIBRARIES ${TCPDUMP_LINK_LIBRARIES} nsl)
    endif(LIBNSL_HAS_GETRPCBYNUMBER)
endif(STDLIBS_HAVE_GETRPCBYNUMBER)
cmake_pop_check_state()

#
# This requires the libraries we require, as ether_ntohost might be
# in one of those libraries.  That means we have to do this after
# we check for those libraries.
#
# You are in a twisty little maze of UN*Xes, all different.
# Some might not have ether_ntohost().
# Some might have it and declare it in <net/ethernet.h>.
# Some might have it and declare it in <netinet/ether.h>
# Some might have it and declare it in <sys/ethernet.h>.
# Some might have it and declare it in <arpa/inet.h>.
# Some might have it and declare it in <netinet/if_ether.h>.
# Some might have it and not declare it in any header file.
#
# Before you is a C compiler.
#
cmake_push_check_state()
set(CMAKE_REQUIRED_LIBRARIES ${TCPDUMP_LINK_LIBRARIES})
check_function_exists(ether_ntohost HAVE_ETHER_NTOHOST)
if(HAVE_ETHER_NTOHOST)
    #
    # OK, we have ether_ntohost().  We don't check whether it's buggy,
    # as we assume any system that has CMake is likely to be new enough
    # that, if it has ether_ntohost(), whatever bug is checked for in
    # autotools is fixed; we just decide to use it.
    #
    set(USE_ETHER_NTOHOST TRUE)

    #
    # Is it declared in <net/ethernet.h>?
    #
    # This test fails if we don't have <net/ethernet.h> or if we do
    # but it doesn't declare ether_ntohost().
    #
    check_symbol_exists(ether_ntohost net/ethernet.h NET_ETHERNET_H_DECLARES_ETHER_NTOHOST)
    if(NET_ETHERNET_H_DECLARES_ETHER_NTOHOST)
        #
        # Yes - we have it declared.
        #
        set(HAVE_DECL_ETHER_NTOHOST TRUE)
    endif()
    #
    # Did that succeed?
    #
    if(NOT HAVE_DECL_ETHER_NTOHOST)
        #
        # No - how about <netinet/ether.h>, as on Linux?
        #
        # This test fails if we don't have <netinet/ether.h>
        # or if we do but it doesn't declare ether_ntohost().
        #
        check_symbol_exists(ether_ntohost netinet/ether.h NETINET_ETHER_H_DECLARES_ETHER_NTOHOST)
        if(NETINET_ETHER_H_DECLARES_ETHER_NTOHOST)
            #
            # Yes - we have it declared.
            #
            set(HAVE_DECL_ETHER_NTOHOST TRUE)
        endif()
    endif()
    #
    # Did that succeed?
    #
    if(NOT HAVE_DECL_ETHER_NTOHOST)
        #
        # No - how about <sys/ethernet.h>, as on Solaris 10 and later?
        #
        # This test fails if we don't have <sys/ethernet.h>
        # or if we do but it doesn't declare ether_ntohost().
        #
        check_symbol_exists(ether_ntohost sys/ethernet.h SYS_ETHERNET_H_DECLARES_ETHER_NTOHOST)
        if(SYS_ETHERNET_H_DECLARES_ETHER_NTOHOST)
            #
            # Yes - we have it declared.
            #
            set(HAVE_DECL_ETHER_NTOHOST TRUE)
        endif()
    endif()
    #
    # Did that succeed?
    #
    if(NOT HAVE_DECL_ETHER_NTOHOST)
        #
        # No, how about <arpa/inet.h>, as on AIX?
        #
        # This test fails if we don't have <arpa/inet.h>
        # or if we do but it doesn't declare ether_ntohost().
        #
        check_symbol_exists(ether_ntohost arpa/inet.h ARPA_INET_H_DECLARES_ETHER_NTOHOST)
        if(ARPA_INET_H_DECLARES_ETHER_NTOHOST)
            #
            # Yes - we have it declared.
            #
            set(HAVE_DECL_ETHER_NTOHOST TRUE)
        endif()
    endif()
    #
    # Did that succeed?
    #
    if(NOT HAVE_DECL_ETHER_NTOHOST)
        #
        # No, how about <netinet/if_ether.h>?
        # On some platforms, it requires <net/if.h> and
        # <netinet/in.h>, and we always include it with
        # both of them, so test it with both of them.
        #
        # This test fails if we don't have <netinet/if_ether.h>
        # and the headers we include before it, or if we do but
        # <netinet/if_ether.h> doesn't declare ether_ntohost().
        #
        check_symbol_exists(ether_ntohost "sys/types.h;sys/socket.h;net/if.h;netinet/in.h;netinet/if_ether.h" NETINET_IF_ETHER_H_DECLARES_ETHER_NTOHOST)
        if(NETINET_IF_ETHER_H_DECLARES_ETHER_NTOHOST)
            #
            # Yes - we have it declared.
            #
            set(HAVE_DECL_ETHER_NTOHOST TRUE)
        endif()
    endif()
    #
    # After all that, is ether_ntohost() declared?
    #
    if(NOT HAVE_DECL_ETHER_NTOHOST)
        #
        # No, we'll have to declare it ourselves.
        # Do we have "struct ether_addr" if we include<netinet/if_ether.h>?
        #
        check_struct_has_member("struct ether_addr" octet "sys/types.h;sys/socket.h;net/if.h;netinet/in.h;netinet/if_ether.h" HAVE_STRUCT_ETHER_ADDR)
    endif()
endif()
cmake_pop_check_state()

check_function_exists(dnet_htoa STDLIBS_HAVE_DNET_HTOA)
if(STDLIBS_HAVE_DNET_HTOA)
    set(HAVE_DNET_HTOA TRUE)
else(STDLIBS_HAVE_DNET_HTOA)
    check_library_exists(dnet dnet_htoa "" HAVE_DNET_HTOA)
    if(HAVE_DNET_HTOA)
        set(TCPDUMP_LINK_LIBRARIES dnet ${TCPDUMP_LINK_LIBRARIES})
    endif(HAVE_DNET_HTOA)
endif(STDLIBS_HAVE_DNET_HTOA)
if(HAVE_DNET_HTOA)
    #
    # OK, we have dnet_htoa().  Do we have netdnet/dnetdb.h?
    #
    check_include_files("sys/types.h;netdnet/dnetdb.h" HAVE_NETDNET_DNETDB_H)
    if(HAVE_NETDNET_DNETDB_H)
        #
        # Yes.  Does it declare dnet_htoa()?
        #
        cmake_push_check_state()
        set(CMAKE_REQUIRED_LIBRARIES dnet)
        check_symbol_exists(dnet_htoa "sys/types.h;netdnet/dnetdb.h" NETDNET_DNETDB_H_DECLARES_DNET_HTOA)
        cmake_pop_check_state()
    endif(HAVE_NETDNET_DNETDB_H)

    #
    # Do we have netdnet/dn.h?
    #
    check_include_file(netdnet/dn.h HAVE_NETDNET_DN_H)
    if(HAVE_NETDNET_DN_H)
        #
        # Yes.  Does it declare struct dn_naddr?
        #
        cmake_push_check_state()
        set(CMAKE_EXTRA_INCLUDE_FILES netdnet/dn.h)
        check_type_size("struct dn_naddr" STRUCT_DN_NADDR)
        cmake_pop_check_state()
    endif(HAVE_NETDNET_DN_H)
endif(HAVE_DNET_HTOA)

#
# Data types.
#
# XXX - there's no check_struct() macro that's like check_struct_has_member()
# except that it only checks for the existence of the structure type,
# so we use check_struct_has_member() and look for ss_family.
#

#
# Check for IPv6 support.
# We just check for AF_INET6 and struct in6_addr.
#
cmake_push_check_state()
if(WIN32)
    set(CMAKE_EXTRA_INCLUDE_FILES sys/types.h ws2tcpip.h)
    check_symbol_exists(AF_INET6 "sys/types.h;ws2tcpip.h" HAVE_AF_INET6)
else(WIN32)
    set(CMAKE_EXTRA_INCLUDE_FILES sys/types.h sys/socket.h netinet/in.h)
    check_symbol_exists(AF_INET6 "sys/types.h;sys/socket.h;netinet/in.h" HAVE_AF_INET6)
endif(WIN32)
check_type_size("struct in6_addr" HAVE_STRUCT_IN6_ADDR)
cmake_pop_check_state()
if(HAVE_AF_INET6 AND HAVE_STRUCT_IN6_ADDR)
    set(HAVE_OS_IPV6_SUPPORT TRUE)
endif(HAVE_AF_INET6 AND HAVE_STRUCT_IN6_ADDR)

######################################
# External dependencies
######################################

#
# libpcap/WinPcap.
# First, find it.
#
find_package(PCAP REQUIRED)
include_directories(${PCAP_INCLUDE_DIRS})

cmake_push_check_state()

#
# Now check headers.
#
set(CMAKE_REQUIRED_INCLUDES ${PCAP_INCLUDE_DIRS})

#
# Check whether we have pcap/pcap-inttypes.h.
# If we do, we use that to get the C99 types defined.
#
check_include_file(pcap/pcap-inttypes.h HAVE_PCAP_PCAP_INTTYPES_H)

#
# Check for various functions in libpcap/WinPcap.
#
cmake_push_check_state()
set(CMAKE_REQUIRED_LIBRARIES ${PCAP_LIBRARIES})

#
# Check for "pcap_list_datalinks()" and use a substitute version if
# it's not present.  If it is present, check for "pcap_free_datalinks()";
# if it's not present, we don't replace it for now.  (We could do so
# on UN*X, but not on Windows, where hilarity ensues if a program
# built with one version of the MSVC support library tries to free
# something allocated by a library built with another version of
# the MSVC support library.)
#
check_function_exists(pcap_list_datalinks HAVE_PCAP_LIST_DATALINKS)
if(HAVE_PCAP_LIST_DATALINKS)
    check_function_exists(pcap_free_datalinks HAVE_PCAP_FREE_DATALINKS)
endif(HAVE_PCAP_LIST_DATALINKS)

#
# Check for "pcap_datalink_name_to_val()", and use a substitute
# version if it's not present.  If it is present, check for
# "pcap_datalink_val_to_description()", and if we don't have it,
# use a substitute version.
#
check_function_exists(pcap_datalink_name_to_val HAVE_PCAP_DATALINK_NAME_TO_VAL)
if(HAVE_PCAP_DATALINK_NAME_TO_VAL)
    check_function_exists(pcap_datalink_val_to_description HAVE_PCAP_DATALINK_VAL_TO_DESCRIPTION)
endif(HAVE_PCAP_DATALINK_NAME_TO_VAL)

#
# Check for "pcap_set_datalink()"; you can't substitute for it if
# it's absent (it has hooks into libpcap), so just define the
# HAVE_ value if it's there.
#
check_function_exists(pcap_set_datalink HAVE_PCAP_SET_DATALINK)

#
# Check for "pcap_breakloop()"; you can't substitute for it if
# it's absent (it has hooks into the live capture routines),
# so just define the HAVE_ value if it's there.
#
check_function_exists(pcap_breakloop HAVE_PCAP_BREAKLOOP)

#
# Check for "pcap_dump_ftell()"; we use a substitute version
# if it's not present.
#
check_function_exists(pcap_dump_ftell HAVE_PCAP_DUMP_FTELL)

#
# Do we have the new open API?  Check for pcap_create() and for
# pcap_statustostr(), and assume that, if we have both of them,
# we also have pcap_activate() and the other new routines
# introduced in libpcap 1.0.0.  (We check for pcap_statustostr()
# as well, because WinPcap 4.1.3 screwed up and exported pcap_create()
# but not other routines such as pcap_statustostr(), even though it
# defined them and even though you really want pcap_statustostr() to
# get strings corresponding to some of the status returns from the
# new routines.)
#
check_function_exists(pcap_statustostr HAVE_PCAP_STATUSTOSTR)
#
# If we don't have pcap_statustostr(), don't check for pcap_create(),
# so we pretend we don't have it.
#
if(HAVE_PCAP_STATUSTOSTR)
    check_function_exists(pcap_create HAVE_PCAP_CREATE)
endif(HAVE_PCAP_STATUSTOSTR)
if(HAVE_PCAP_CREATE)
    #
    # OK, do we have pcap_set_tstamp_type?  If so, assume we have
    # pcap_list_tstamp_types and pcap_free_tstamp_types as well.
    #
    check_function_exists(pcap_set_tstamp_type HAVE_PCAP_SET_TSTAMP_TYPE)

    #
    # And do we have pcap_set_tstamp_precision?  If so, we assume
    # we also have pcap_open_offline_with_tstamp_precision.
    #
    check_function_exists(pcap_set_tstamp_precision HAVE_PCAP_SET_TSTAMP_PRECISION)
endif(HAVE_PCAP_CREATE)

#
# Check for a miscellaneous collection of functions which we use
# if we have them.
#
check_function_exists(pcap_findalldevs HAVE_PCAP_FINDALLDEVS)
if(HAVE_PCAP_FINDALLDEVS)
    #
    # Check for libpcap having pcap_findalldevs() but the pcap.h header
    # not having pcap_if_t; some versions of Mac OS X shipped with pcap.h
    # from 0.6 and libpcap 0.8, so that libpcap had pcap_findalldevs but
    # pcap.h didn't have pcap_if_t.
    #
    cmake_push_check_state()
    set(CMAKE_REQUIRED_INCLUDES ${PCAP_INCLUDE_DIRS})
    set(CMAKE_EXTRA_INCLUDE_FILES pcap.h)
    check_type_size(pcap_if_t PCAP_IF_T)
    cmake_pop_check_state()
endif(HAVE_PCAP_FINDALLDEVS)
check_function_exists(pcap_dump_flush HAVE_PCAP_DUMP_FLUSH)
check_function_exists(pcap_lib_version HAVE_PCAP_LIB_VERSION)
if(NOT HAVE_PCAP_LIB_VERSION)
    # Check for the pcap_version string variable and set HAVE_PCAP_VERSION
endif(NOT HAVE_PCAP_LIB_VERSION)
check_function_exists(pcap_setdirection HAVE_PCAP_SETDIRECTION)
check_function_exists(pcap_set_immediate_mode HAVE_PCAP_SET_IMMEDIATE_MODE)
check_function_exists(pcap_dump_ftell64 HAVE_PCAP_DUMP_FTELL64)
check_function_exists(pcap_open HAVE_PCAP_OPEN)
check_function_exists(pcap_findalldevs_ex HAVE_PCAP_FINDALLDEVS_EX)

#
# On Windows, check for pcap_wsockinit(); if we don't have it, check for
# wsockinit().
#
if(WIN32)
    check_function_exists(pcap_wsockinit HAVE_PCAP_WSOCKINIT)
    if(NOT HAVE_PCAP_WSOCKINIT)
        check_function_exists(wsockinit HAVE_WSOCKINIT)
    endif(NOT HAVE_PCAP_WSOCKINIT)
endif(WIN32)

#
# Check for special debugging functions
#
check_function_exists(pcap_set_parser_debug HAVE_PCAP_SET_PARSER_DEBUG)
if(NOT HAVE_PCAP_SET_PARSER_DEBUG)
    # Check whether libpcap defines pcap_debug or yydebug
    check_variable_exists(pcap_debug HAVE_PCAP_DEBUG)
    if(NOT HAVE_PCAP_DEBUG)
        check_variable_exists(yydebug HAVE_YYDEBUG)
    endif(NOT HAVE_PCAP_DEBUG)
endif(NOT HAVE_PCAP_SET_PARSER_DEBUG)

check_function_exists(pcap_set_optimizer_debug HAVE_PCAP_SET_OPTIMIZER_DEBUG)
check_function_exists(bpf_dump HAVE_BPF_DUMP)

cmake_pop_check_state()

#
# We have libpcap.
#
include_directories(SYSTEM ${PCAP_INCLUDE_DIRS})
set(TCPDUMP_LINK_LIBRARIES ${PCAP_LIBRARIES} ${TCPDUMP_LINK_LIBRARIES})

#
# Optional libraries.
#

#
# libsmi.
#
if(WITH_SMI)
    find_package(SMI)
    if(SMI_FOUND)
        include_directories(SYSTEM ${SMI_INCLUDE_DIRS})
        set(TCPDUMP_LINK_LIBRARIES ${TCPDUMP_LINK_LIBRARIES} ${SMI_LIBRARIES})
        set(USE_LIBSMI ON)
    endif(SMI_FOUND)
endif(WITH_SMI)

#
# OpenSSL/libressl libcrypto.
#
if(WITH_CRYPTO)
    find_package(CRYPTO)
    if(CRYPTO_FOUND)
        #
        # Check for some functions.
        #
        cmake_push_check_state()
        set(CMAKE_REQUIRED_LIBRARIES crypto)

        #
      	# 1) do we have EVP_CIPHER_CTX_new?
        # If so, we use it to allocate an EVP_CIPHER_CTX, as
        # EVP_CIPHER_CTX may be opaque; otherwise, we allocate
        # it ourselves.
        #
        check_function_exists(EVP_CIPHER_CTX_new HAVE_EVP_CIPHER_CTX_NEW)

        #
        # 2) do we have EVP_CipherInit_ex()?
        # If so, we use it, because we need to be able to make two
        # "initialize the cipher" calls, one with the cipher and key,
        # and one with the IV, and, as of OpenSSL 1.1, You Can't Do That
        # with EVP_CipherInit(), because a call to EVP_CipherInit() will
        # unconditionally clear the context, and if you don't supply a
        # cipher, it'll clear the cipher, rendering the context unusable
        # and causing a crash.
        #
        check_function_exists(EVP_CipherInit_ex HAVE_EVP_CIPHERINIT_EX)

        cmake_pop_check_state()

        #
        # We have libcrypto.
        #
        include_directories(SYSTEM ${CRYPTO_INCLUDE_DIRS})
        set(TCPDUMP_LINK_LIBRARIES ${TCPDUMP_LINK_LIBRARIES} ${CRYPTO_LIBRARIES})
        set(HAVE_LIBCRYPTO ON)
    endif(CRYPTO_FOUND)
endif(WITH_CRYPTO)

#
# Capsicum sandboxing.
# Some of this is in the system library, some of it is in other libraries.
#
if(WITH_CAPSICUM)
    check_function_exists(cap_enter HAVE_CAP_ENTER)
    check_function_exists(cap_rights_limit HAVE_CAP_RIGHTS_LIMIT)
    check_function_exists(cap_ioctls_limit HAVE_CAP_IOCTLS_LIMIT)
    check_function_exists(openat HAVE_OPENAT)
    if(HAVE_CAP_ENTER AND HAVE_CAP_RIGHTS_LIMIT AND
       HAVE_CAP_IOCTLS_LIMIT AND HAVE_OPENAT)
        #
        # OK, we have the functions we need to support Capsicum.
        #
        set(HAVE_CAPSICUM TRUE)

        #
        # OK, can we use Casper?
        #
        check_library_exists(casper cap_init "" HAVE_CAP_INIT)
        if(HAVE_CAP_INIT)
            cmake_push_check_state()
            set(CMAKE_REQUIRED_LIBRARIES casper)
            check_library_exists(cap_dns cap_gethostbyaddr "" HAVE_CAP_GETHOSTBYADDR)
            cmake_pop_check_state()
            if(HAVE_CAP_GETHOSTBYADDR)
                set(HAVE_CASPER TRUE)
                set(TCPDUMP_LINK_LIBRARIES ${TCPDUMP_LINK_LIBRARIES} casper cap_dns)
            endif(HAVE_CAP_GETHOSTBYADDR)
        endif(HAVE_CAP_INIT)
    endif(HAVE_CAP_ENTER AND HAVE_CAP_RIGHTS_LIMIT AND
          HAVE_CAP_IOCTLS_LIMIT AND HAVE_OPENAT)
endif(WITH_CAPSICUM)

#
# libcap-ng.
#
if(WITH_CAP_NG)
    check_include_file(cap-ng.h HAVE_CAP_NG_H)
    check_library_exists(cap-ng capng_change_id "" HAVE_LIBCAP_NG)
    if(HAVE_LIBCAP_NG)
        set(TCPDUMP_LINK_LIBRARIES ${TCPDUMP_LINK_LIBRARIES} cap-ng)
    endif(HAVE_LIBCAP_NG)
endif(WITH_CAP_NG)

###################################################################
#   Warning options
###################################################################

#
# Check and add warning options if we have a .devel file.
#
if(EXISTS ${CMAKE_SOURCE_DIR}/.devel OR EXISTS ${CMAKE_BINARY_DIR}/.devel)
    #
    # Warning options.
    #
    if(MSVC AND NOT ${CMAKE_C_COMPILER} MATCHES "clang*")
        #
        # MSVC, with Microsoft's front end and code generator.
        # "MSVC" is also set for Microsoft's compiler with a Clang
        # front end and their code generator ("Clang/C2"), so we
        # check for clang.exe and treat that differently.
        #
        check_and_add_compiler_option(-Wall)
        #
        # Disable some pointless warnings that /Wall turns on.
        #
        # Unfortunately, MSVC does not appear to have an equivalent
        # to "__attribute__((unused))" to mark a particular function
        # parameter as being known to be unused, so that the compiler
        # won't warn about it (for example, the function might have
        # that parameter because a pointer to it is being used, and
        # the signature of that function includes that parameter).
        # C++ lets you give a parameter a type but no name, but C
        # doesn't have that.
        #
        check_and_add_compiler_option(-wd4100)
        #
        # In theory, we care whether somebody uses f() rather than
        # f(void) to declare a function with no arguments, but, in
        # practice, there are places in the Windows header files
        # that appear to do that, so we squelch that warning.
        #
        check_and_add_compiler_option(-wd4255)
        #
        # Windows FD_SET() generates this, so we suppress it.
        #
        check_and_add_compiler_option(-wd4548)
        #
        # Perhaps testing something #defined to be 0 with #ifdef is an
        # error, and it should be tested with #if, but perhaps it's
        # not, and Microsoft does that in its headers, so we squelch
        # that warning.
        #
        check_and_add_compiler_option(-wd4574)
        #
        # The Windows headers also test not-defined values in #if, so
        # we don't want warnings about that, either.
        #
        check_and_add_compiler_option(-wd4668)
        #
        # We do *not* care whether some function is, or isn't, going to be
        # expanded inline.
        #
        check_and_add_compiler_option(-wd4710)
        check_and_add_compiler_option(-wd4711)
        #
        # We do *not* care whether we're adding padding bytes after
        # structure members.
        #
        check_and_add_compiler_option(-wd4820)
    else()
        #
        # Other compilers, including MSVC with a Clang front end and
        # Microsoft's code generator.  We currently treat them as if
        # they might support GCC-style -W options.
        #
        check_and_add_compiler_option(-W)
        check_and_add_compiler_option(-Wall)
        check_and_add_compiler_option(-Wassign-enum)
        check_and_add_compiler_option(-Wcast-qual)
        check_and_add_compiler_option(-Wmissing-prototypes)
        check_and_add_compiler_option(-Wold-style-definition)
        check_and_add_compiler_option(-Wpedantic)
        check_and_add_compiler_option(-Wpointer-arith)
        check_and_add_compiler_option(-Wshadow)
        check_and_add_compiler_option(-Wsign-compare)
        check_and_add_compiler_option(-Wstrict-prototypes)
        check_and_add_compiler_option(-Wunreachable-code-return)
        check_and_add_compiler_option(-Wused-but-marked-unused)
        check_and_add_compiler_option(-Wwrite-strings)
    endif()
endif()

######################################
# Input files
######################################

if(ENABLE_SMB)
    #
    # We allow the SMB dissector to be omitted.
    #
    set(LOCALSRC ${LOCALSRC}
        print-smb.c
        smbutil.c)
endif(ENABLE_SMB)

set(NETDISSECT_SOURCE_LIST_C
    addrtoname.c
    addrtostr.c
    af.c
    ascii_strcasecmp.c
    checksum.c
    cpack.c
    gmpls.c
    in_cksum.c
    ipproto.c
    l2vpn.c
    machdep.c
    netdissect.c
    netdissect-alloc.c
    nlpid.c
    oui.c
    parsenfsfh.c
    print.c
    print-802_11.c
    print-802_15_4.c
    print-ah.c
    print-ahcp.c
    print-aodv.c
    print-aoe.c
    print-ap1394.c
    print-arcnet.c
    print-arp.c
    print-ascii.c
    print-atalk.c
    print-atm.c
    print-babel.c
    print-beep.c
    print-bfd.c
    print-bgp.c
    print-bootp.c
    print-bt.c
    print-calm-fast.c
    print-carp.c
    print-cdp.c
    print-cfm.c
    print-chdlc.c
    print-cip.c
    print-cnfp.c
    print-dccp.c
    print-decnet.c
    print-dhcp6.c
    print-domain.c
    print-dtp.c
    print-dvmrp.c
    print-eap.c
    print-egp.c
    print-eigrp.c
    print-enc.c
    print-esp.c
    print-ether.c
    print-fddi.c
    print-forces.c
    print-fr.c
    print-frag6.c
    print-ftp.c
    print-geneve.c
    print-geonet.c
    print-gre.c
    print-hncp.c
    print-hsrp.c
    print-http.c
    print-icmp.c
    print-icmp6.c
    print-igmp.c
    print-igrp.c
    print-ip.c
    print-ip6.c
    print-ip6opts.c
    print-ipcomp.c
    print-ipfc.c
    print-ipnet.c
    print-ipx.c
    print-isakmp.c
    print-isoclns.c
    print-juniper.c
    print-krb.c
    print-l2tp.c
    print-lane.c
    print-ldp.c
    print-lisp.c
    print-llc.c
    print-lldp.c
    print-lmp.c
    print-loopback.c
    print-lspping.c
    print-lwapp.c
    print-lwres.c
    print-m3ua.c
    print-medsa.c
    print-mobile.c
    print-mobility.c
    print-mpcp.c
    print-mpls.c
    print-mptcp.c
    print-msdp.c
    print-msnlb.c
    print-nflog.c
    print-nfs.c
    print-nsh.c
    print-ntp.c
    print-null.c
    print-olsr.c
    print-openflow-1.0.c
    print-openflow.c
    print-ospf.c
    print-ospf6.c
    print-otv.c
    print-pgm.c
    print-pim.c
    print-pktap.c
    print-ppi.c
    print-ppp.c
    print-pppoe.c
    print-pptp.c
    print-radius.c
    print-raw.c
    print-resp.c
    print-rip.c
    print-ripng.c
    print-rpki-rtr.c
    print-rrcp.c
    print-rsvp.c
    print-rt6.c
    print-rtsp.c
    print-rx.c
    print-sctp.c
    print-sflow.c
    print-sip.c
    print-sl.c
    print-sll.c
    print-slow.c
    print-smtp.c
    print-snmp.c
    print-stp.c
    print-sunatm.c
    print-sunrpc.c
    print-symantec.c
    print-syslog.c
    print-tcp.c
    print-telnet.c
    print-tftp.c
    print-timed.c
    print-tipc.c
    print-token.c
    print-udld.c
    print-udp.c
    print-usb.c
    print-vjc.c
    print-vqp.c
    print-vrrp.c
    print-vtp.c
    print-vxlan.c
    print-vxlan-gpe.c
    print-wb.c
    print-zephyr.c
    print-zeromq.c
    ${LOCALSRC}
    signature.c
    strtoaddr.c
    util-print.c
)

#
# Replace missing functions
#
foreach(FUNC strlcat strlcpy strdup strsep getservent getopt_long)
    string(TOUPPER ${FUNC} FUNC_UPPERCASE)
    set(HAVE_FUNC_UPPERCASE HAVE_${FUNC_UPPERCASE})
    if(NOT ${HAVE_FUNC_UPPERCASE})
        set(NETDISSECT_SOURCE_LIST_C ${NETDISSECT_SOURCE_LIST_C} missing/${FUNC}.c)
    endif()
endforeach()
if(NOT WIN32)
    if(NOT HAVE_VSNPRINTF OR NOT HAVE_SNPRINTF)
        set(NETDISSECT_SOURCE_LIST_C ${NETDISSECT_SOURCE_LIST_C} missing/snprintf.c)
    endif(NOT HAVE_VSNPRINTF OR NOT HAVE_SNPRINTF)
endif(NOT WIN32)

add_library(netdissect STATIC
    ${NETDISSECT_SOURCE_LIST_C}
)
if(NOT C_ADDITIONAL_FLAGS STREQUAL "")
    set_target_properties(netdissect PROPERTIES COMPILE_FLAGS ${C_ADDITIONAL_FLAGS})
endif()

set(TCPDUMP_SOURCE_LIST_C tcpdump.c)

if(NOT HAVE_BPF_DUMP)
    set(TCPDUMP_SOURCE_LIST_C ${TCPDUMP_SOURCE_LIST_C} bpf_dump.c)
endif(NOT HAVE_BPF_DUMP)
if(NOT HAVE_PCAP_DUMP_FTELL)
    set(TCPDUMP_SOURCE_LIST_C ${TCPDUMP_SOURCE_LIST_C} missing/pcap_dump_ftell.c)
endif(NOT HAVE_PCAP_DUMP_FTELL)

if(NOT HAVE_PCAP_LIST_DATALINKS)
    set(TCPDUMP_SOURCE_LIST_C ${TCPDUMP_SOURCE_LIST_C} missing/datalinks.c)
endif(NOT HAVE_PCAP_LIST_DATALINKS)

if((NOT HAVE_PCAP_DATALINK_NAME_TO_VAL) OR (NOT HAVE_PCAP_DATALINK_VAL_TO_DESCRIPTION))
    set(TCPDUMP_SOURCE_LIST_C ${TCPDUMP_SOURCE_LIST_C} missing/dlnames.c)
endif((NOT HAVE_PCAP_DATALINK_NAME_TO_VAL) OR (NOT HAVE_PCAP_DATALINK_VAL_TO_DESCRIPTION))

set(PROJECT_SOURCE_LIST_C ${NETDISSECT_SOURCE_LIST_C} ${TCPDUMP_SOURCE_LIST_C})

file(GLOB PROJECT_SOURCE_LIST_H
    *.h
)

source_group("Source Files" FILES ${PROJECT_SOURCE_LIST_C})
source_group("Header Files" FILES ${PROJECT_SOURCE_LIST_H})

######################################
# Register targets
######################################

add_executable(tcpdump ${TCPDUMP_SOURCE_LIST_C})
if(NOT C_ADDITIONAL_FLAGS STREQUAL "")
    set_target_properties(tcpdump PROPERTIES COMPILE_FLAGS ${C_ADDITIONAL_FLAGS})
endif()
target_link_libraries(tcpdump netdissect ${TCPDUMP_LINK_LIBRARIES})

######################################
# Write out the config.h file
######################################

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/cmakeconfig.h.in ${CMAKE_CURRENT_BINARY_DIR}/config.h)

######################################
# Install tcpdump and man pages
######################################

#
# "Define GNU standard installation directories", which actually
# are also defined, to some degree, by autotools, and at least
# some of which are general UN*X conventions.
#
include(GNUInstallDirs)

set(MAN1_EXPAND tcpdump.1.in)

if(WIN32)
    # XXX TODO where to install on Windows?
else(WIN32)
    install(TARGETS tcpdump DESTINATION sbin)
endif(WIN32)

# On UN*X, and on Windows when not using MSVC, process man pages and
# arrange that they be installed.
if(NOT MSVC)
    #
    # Man pages.
    #
    # For each section of the manual for which we have man pages
    # that require macro expansion, do the expansion.
    #
    set(MAN1 "")
    foreach(TEMPLATE_MANPAGE ${MAN1_EXPAND})
        string(REPLACE ".in" "" MANPAGE ${TEMPLATE_MANPAGE})
        configure_file(${CMAKE_SOURCE_DIR}/${TEMPLATE_MANPAGE} ${CMAKE_CURRENT_BINARY_DIR}/${MANPAGE} @ONLY)
        set(MAN1 ${MAN1} ${CMAKE_CURRENT_BINARY_DIR}/${MANPAGE})
    endforeach(TEMPLATE_MANPAGE)
    install(FILES ${MAN1} DESTINATION ${CMAKE_INSTALL_MANDIR}/man1)
endif(NOT MSVC)

# uninstall target
configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake_uninstall.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake"
    IMMEDIATE @ONLY)

add_custom_target(uninstall
    COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake)

#
# Tcpdump tests
#
add_custom_target(check
    COMMAND ./TESTrun.sh
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/tests)
