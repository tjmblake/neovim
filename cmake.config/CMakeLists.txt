include(CheckTypeSize)
include(CheckSymbolExists)
include(CheckFunctionExists)
include(CheckIncludeFiles)
include(CheckCSourceRuns)
include(CheckCSourceCompiles)

check_c_source_compiles("
#include <execinfo.h>
int main(void)
{
  void *trace[1];
  backtrace(trace, 1);
  return 0;
}
" HAVE_EXECINFO_BACKTRACE)

check_c_source_compiles("
int main(void)
{
  int a = 42;
  __builtin_add_overflow(a, a, &a);
  __builtin_sub_overflow(a, a, &a);
  return 0;
}
" HAVE_BUILTIN_ADD_OVERFLOW)

check_type_size("int" SIZEOF_INT LANGUAGE C)
check_type_size("long" SIZEOF_LONG LANGUAGE C)
check_type_size("intmax_t" SIZEOF_INTMAX_T LANGUAGE C)
check_type_size("size_t" SIZEOF_SIZE_T LANGUAGE C)
check_type_size("void *" SIZEOF_VOID_PTR LANGUAGE C)

check_symbol_exists(_NSGetEnviron crt_externs.h HAVE__NSGETENVIRON)

# Headers
check_include_files(langinfo.h HAVE_LANGINFO_H)
check_include_files(strings.h HAVE_STRINGS_H)
check_include_files(sys/utsname.h HAVE_SYS_UTSNAME_H)
check_include_files(termios.h HAVE_TERMIOS_H)
check_include_files(sys/uio.h HAVE_SYS_UIO_H)
check_include_files(sys/sdt.h HAVE_SYS_SDT_H)
if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
  check_include_files(sys/xattr.h HAVE_XATTR)
endif()

# Functions
check_function_exists(fseeko HAVE_FSEEKO)
check_function_exists(readv HAVE_READV)
check_function_exists(readlink HAVE_READLINK)
check_function_exists(strnlen HAVE_STRNLEN)
check_function_exists(strcasecmp HAVE_STRCASECMP)
check_function_exists(strncasecmp HAVE_STRNCASECMP)
check_function_exists(strptime HAVE_STRPTIME)

check_c_source_compiles("
#include <sys/types.h>
#include <dirent.h>
#include <sys/file.h>
int main(void)
{
  DIR *dir = opendir(\"dirname\");
  dirfd(dir);
  flock(10, LOCK_SH);
  return 0;
}
" HAVE_DIRFD_AND_FLOCK)


check_c_source_compiles("
#include <pwd.h>
int main(void)
{
  getpwent();
  getpwuid(0);
  getpwnam(\"root\");
  return 0;
}
" HAVE_PWD_FUNCS)


if(CMAKE_SYSTEM_NAME STREQUAL "SunOS")
  check_c_source_compiles("
#include <termios.h>
int
main(void)
{
  return forkpty(0, NULL, NULL, NULL);
}
" HAVE_FORKPTY)
else()
  set(HAVE_FORKPTY 1)
endif()

# Symbols
check_symbol_exists(FD_CLOEXEC "fcntl.h" HAVE_FD_CLOEXEC)
if(HAVE_LANGINFO_H)
  check_symbol_exists(CODESET "langinfo.h" HAVE_NL_LANGINFO_CODESET)
endif()

check_include_files("endian.h" HAVE_ENDIAN_H)

set(ENDIAN_INCLUDE_FILE "endian.h")
if(NOT HAVE_ENDIAN_H)
  check_include_files("sys/endian.h" HAVE_SYS_ENDIAN_H)
  if (HAVE_SYS_ENDIAN_H)
    set(ENDIAN_INCLUDE_FILE "sys/endian.h")
  endif()
endif()

set(SI "#include <stdint.h>\n")
set(MS "int main(int argc,char**argv)\n{\n  uint64_t i=0x0102030405060708ULL;")
set(ME "}")
check_c_source_compiles("
  #define _BSD_SOURCE 1
  #define _DEFAULT_SOURCE 1
  ${SI}
  #include <${ENDIAN_INCLUDE_FILE}>
  #ifndef be64toh
  # error No be64toh macros
  #endif
  ${MS}
    uint64_t j = be64toh(i);
    return (j == 0);  // j must not be zero
  ${ME}"
  HAVE_BE64TOH_MACROS)
if(NOT "${HAVE_BE64TOH_MACROS}")
  check_function_exists(be64toh HAVE_BE64TOH_FUNC)
endif()
if("${HAVE_BE64TOH_MACROS}" OR "${HAVE_BE64TOH_FUNC}")
  set(HAVE_BE64TOH 1)
endif()
if (NOT "${HAVE_BE64TOH}")
  if (NOT "${CMAKE_CROSSCOMPILING}")
    # It is safe to make ORDER_BIG_ENDIAN not defined if
    # - HAVE_BE64TOH is true. In this case be64toh will be used unconditionally in
    #   any case and ORDER_BIG_ENDIAN will not be examined.
    # - CMAKE_CROSSCOMPILING *and* HAVE_BE64TOH are both false. In this case
    #   be64toh function which uses cycle and arithmetic operations is used which
    #   will work regardless of endianness. Function is sub-optimal though.
    check_c_source_runs("
      ${SI}
      ${MS}
        char *s = (char *) &i;
        return (
          s[0] == 0x01
          && s[1] == 0x02
          && s[2] == 0x03
          && s[3] == 0x04
          && s[4] == 0x05
          && s[5] == 0x06
          && s[6] == 0x07
          && s[7] == 0x08) ? 0 : 1;
      ${ME}"
      ORDER_BIG_ENDIAN)
  endif()
endif()

configure_file (
  "${PROJECT_SOURCE_DIR}/cmake.config/config.h.in"
  "${PROJECT_BINARY_DIR}/cmake.config/auto/config.h"
  )

# Prevent double space in --version output if CMAKE_C_FLAGS is empty
set(C_FLAGS_VERSION_OUTPUT ${CMAKE_C_FLAGS})
if(C_FLAGS_VERSION_OUTPUT)
  string(PREPEND C_FLAGS_VERSION_OUTPUT " ")
endif()

if(${CMAKE_VERSION} VERSION_GREATER_EQUAL 3.15)
  configure_file(versiondef.h.in auto/versiondef.h.gen)
else()
  configure_file(versiondef_old.h.in auto/versiondef.h.gen)
endif()

file(GENERATE
  OUTPUT "${PROJECT_BINARY_DIR}/cmake.config/auto/versiondef-$<CONFIG>.h"
  INPUT "${PROJECT_BINARY_DIR}/cmake.config/auto/versiondef.h.gen")

configure_file (
  "${PROJECT_SOURCE_DIR}/cmake.config/pathdef.c.in"
  "${PROJECT_BINARY_DIR}/cmake.config/auto/pathdef.c"
  ESCAPE_QUOTES)
