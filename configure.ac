AC_INIT([cpuminer], [2.4.3])

AC_PREREQ([2.59c])
AC_CANONICAL_SYSTEM
AC_CONFIG_MACRO_DIR([m4])
AC_CONFIG_SRCDIR([cpu-miner.c])
AM_INIT_AUTOMAKE([gnu subdir-objects])
AC_CONFIG_HEADERS([cpuminer-config.h])
# AM_INIT_AUTOMAKE([subdir-objects])

dnl Make sure anyone changing configure.ac/Makefile.am has a clue
AM_MAINTAINER_MODE

dnl Checks for programs
AC_PROG_CC_C99
AC_PROG_GCC_TRADITIONAL
AM_PROG_CC_C_O
AM_PROG_AS
LT_INIT([disable-shared])

dnl Checks for header files
AC_HEADER_STDC
AC_CHECK_HEADERS([sys/endian.h sys/param.h syslog.h])
# sys/sysctl.h requires sys/types.h on FreeBSD
# sys/sysctl.h requires sys/param.h on OpenBSD
AC_CHECK_HEADERS([sys/sysctl.h], [], [],
[#include <sys/types.h>
#ifdef HAVE_SYS_PARAM_H
#include <sys/param.h>
#endif
])

AC_CHECK_DECLS([be32dec, le32dec, be32enc, le32enc], [], [],
[AC_INCLUDES_DEFAULT
#ifdef HAVE_SYS_ENDIAN_H
#include <sys/endian.h>
#endif
])

AC_FUNC_ALLOCA
AC_CHECK_FUNCS([getopt_long])

case $target in
  i*86-*-*)
    have_x86=true
    ;;
  x86_64-*-*|amd64-*-*)
    have_x86_64=true
    ;;
  arm*-*-*)
    have_arm=true
    ;;
  powerpc*-*-*)
    have_ppc=true
    ;;
esac

PTHREAD_FLAGS="-pthread"
WS2_LIBS=""

case $target in
  *-*-mingw*)
    have_win32=true
    PTHREAD_FLAGS=""
    WS2_LIBS="-lws2_32"
    ;;
esac

have_sdk_dir=false
if test -n "$SDK_DIR"; then
    have_sdk_dir=true
    CPPFLAGS="-I$SDK_DIR/include $CPPFLAGS"
    LDFLAGS="-L$SDK_DIR/lib/$target $LDFLAGS"
fi

AC_ARG_ENABLE([assembly],
  AS_HELP_STRING([--disable-assembly], [disable assembly-language routines]))
if test x$enable_assembly != xno; then
  AC_DEFINE([USE_ASM], [1], [Define to 1 if assembly routines are wanted.])
fi

if test x$enable_assembly != xno -a x$have_x86_64 = xtrue
then
  AC_MSG_CHECKING(whether we can compile AVX code)
  AC_COMPILE_IFELSE([AC_LANG_PROGRAM(,[asm ("vmovdqa %ymm0, %ymm1");])],
    AC_DEFINE(USE_AVX, 1, [Define to 1 if AVX assembly is available.])
    AC_MSG_RESULT(yes)
    AC_MSG_CHECKING(whether we can compile XOP code)
    AC_COMPILE_IFELSE([AC_LANG_PROGRAM(,[asm ("vprotd \$7, %xmm0, %xmm1");])],
      AC_DEFINE(USE_XOP, 1, [Define to 1 if XOP assembly is available.])
      AC_MSG_RESULT(yes)
    ,
      AC_MSG_RESULT(no)
      AC_MSG_WARN([The assembler does not support the XOP instruction set.])
    )
    AC_MSG_CHECKING(whether we can compile AVX2 code)
    AC_COMPILE_IFELSE([AC_LANG_PROGRAM(,[asm ("vpaddd %ymm0, %ymm1, %ymm2");])],
      AC_DEFINE(USE_AVX2, 1, [Define to 1 if AVX2 assembly is available.])
      AC_MSG_RESULT(yes)
    ,
      AC_MSG_RESULT(no)
      AC_MSG_WARN([The assembler does not support the AVX2 instruction set.])
    )
  ,
    AC_MSG_RESULT(no)
    AC_MSG_WARN([The assembler does not support the AVX instruction set.])
  )
fi

AC_CHECK_LIB(jansson, json_loads, request_jansson=false, request_jansson=true)
AC_CHECK_LIB([pthread], [pthread_create], PTHREAD_LIBS="-lpthread",
  AC_CHECK_LIB([pthreadGC2], [pthread_create], PTHREAD_LIBS="-lpthreadGC2",
    AC_CHECK_LIB([pthreadGC1], [pthread_create], PTHREAD_LIBS="-lpthreadGC1",
      AC_CHECK_LIB([pthreadGC], [pthread_create], PTHREAD_LIBS="-lpthreadGC"
))))

AC_ARG_WITH([build_number],
    [AC_HELP_STRING([--with-build-number], [Specify a build number.])],
    [BUILD_NUMBER="$withval"]
    )
AC_SUBST([BUILD_NUMBER])
AM_CONDITIONAL([USE_BUILD_NUMBER], [test x$BUILD_NUMBER != x])

AM_CONDITIONAL([WANT_JANSSON], [test x$request_jansson = xtrue])
AM_CONDITIONAL([HAVE_WINDOWS], [test x$have_win32 = xtrue])
AM_CONDITIONAL([USE_ASM], [test x$enable_assembly != xno])
AM_CONDITIONAL([ARCH_x86], [test x$have_x86 = xtrue])
AM_CONDITIONAL([ARCH_x86_64], [test x$have_x86_64 = xtrue])
AM_CONDITIONAL([ARCH_ARM], [test x$have_arm = xtrue])
AM_CONDITIONAL([ARCH_PPC], [test x$have_ppc = xtrue])

if test x$request_jansson = xtrue
then
	JANSSON_LIBS="compat/jansson/libjansson.a"
else
	JANSSON_LIBS=-ljansson
fi

PKG_PROG_PKG_CONFIG()

if test "x$have_sdk_dir" = "xtrue"; then
    if test "x$have_x86_64" = xtrue; then
        ARCH_DIR=x86_64
    else
        ARCH_DIR=x86
    fi
    echo PKG_CONFIG_PATH
    PKG_CONFIG="${PKG_CONFIG:-pkg-config} --define-variable=arch=$ARCH_DIR --define-variable=target=$target --define-variable=sgminersdkdir=$SDK_DIR"
    PKG_CONFIG_PATH="$SDK_DIR/lib/pkgconfig${PKG_CONFIG_PATH:+:$PKG_CONFIG_PATH}"
    export PKG_CONFIG_PATH
fi

PKG_CHECK_MODULES([OPENSSL], [openssl], ,[AC_MSG_ERROR([Missing required openssl dev])])

if test "x$have_win32" != xtrue; then
    PKG_CHECK_MODULES([LIBCURL], [libcurl >= 7.25.0], [AC_DEFINE([CURL_HAS_KEEPALIVE], [1], [Defined if version of curl supports keepalive.])],
    [PKG_CHECK_MODULES([LIBCURL], [libcurl >= 7.15.2], ,[AC_MSG_ERROR([Missing required libcurl dev >= 7.15.2])])])
else
    PKG_CHECK_MODULES([LIBCURL], [libcurl >= 7.25.0], ,[AC_MSG_ERROR([Missing required libcurl dev >= 7.25.0])])
    AC_DEFINE([CURL_HAS_KEEPALIVE], [1])
fi
AC_DEFINE([HAVE_LIBCURL], [1], [Defined to 1 if libcurl support built in])

AC_SUBST(LIBCURL_CFLAGS)
AC_SUBST(LIBCURL_CPPFLAGS)
AC_SUBST(LIBCURL_LIBS)
AC_SUBST(OPENSSL_LIBS)
AC_SUBST(JANSSON_LIBS)
AC_SUBST(PTHREAD_FLAGS)
AC_SUBST(PTHREAD_LIBS)
AC_SUBST(WS2_LIBS)

AC_CONFIG_FILES([
	Makefile
	compat/Makefile
	compat/jansson/Makefile
	])
AC_OUTPUT
