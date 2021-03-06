ExternalProject_Add(binutils
    URL "http://ftp.gnu.org/gnu/binutils/binutils-2.26.tar.bz2"
    URL_HASH SHA256=c2ace41809542f5237afc7e3b8f32bb92bc7bc53c6232a84463c423b0714ecd9
    CONFIGURE_COMMAND <SOURCE_DIR>/configure
        --target=${TARGET_ARCH}
        --prefix=${CMAKE_INSTALL_PREFIX}
        --with-sysroot=${CMAKE_INSTALL_PREFIX}
        --disable-multilib
        --disable-nls
        --disable-shared
    BUILD_COMMAND make -j${MAKEJOBS}
    INSTALL_COMMAND make install
    LOG_DOWNLOAD 1 LOG_CONFIGURE 1 LOG_BUILD 1 LOG_INSTALL 1
)

find_program(PKGCONFIG NAMES pkg-config)

ExternalProject_Add_Step(binutils basedirs
    DEPENDEES download
    COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_INSTALL_PREFIX}
    COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_INSTALL_PREFIX}/bin
    COMMAND ${CMAKE_COMMAND} -E create_symlink ${PKGCONFIG} ${CMAKE_INSTALL_PREFIX}/bin/${TARGET_ARCH}-pkg-config
    COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_INSTALL_PREFIX}/${TARGET_ARCH}
    COMMAND ${CMAKE_COMMAND} -E create_symlink ${CMAKE_INSTALL_PREFIX}/${TARGET_ARCH} ${CMAKE_INSTALL_PREFIX}/mingw
    COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_INSTALL_PREFIX}/${TARGET_ARCH}/lib
    COMMAND ${CMAKE_COMMAND} -E create_symlink ${CMAKE_INSTALL_PREFIX}/${TARGET_ARCH}/lib ${CMAKE_INSTALL_PREFIX}/${TARGET_ARCH}/lib64
    COMMENT "Setting up target directories and symlinks"
)

ExternalProject_Add(mingw-w64-headers
    PREFIX mingw-w64-prefix
    STAMP_DIR mingw-w64-prefix/src/mingw-w64-stamp
    SOURCE_DIR mingw-w64-prefix/src/mingw-w64
    BINARY_DIR mingw-w64-prefix/src/mingw-w64-headers-build
    DEPENDS binutils
    GIT_REPOSITORY git://git.code.sf.net/p/mingw-w64/mingw-w64
    UPDATE_COMMAND ""
    PATCH_COMMAND ${EXEC} git am ${CMAKE_CURRENT_SOURCE_DIR}/mingw-w64-*.patch
    CONFIGURE_COMMAND <SOURCE_DIR>/mingw-w64-headers/configure
        --host=${TARGET_ARCH}
        --prefix=${MINGW_INSTALL_PREFIX}
        --enable-sdk=all
        --enable-secure-api
    BUILD_COMMAND ""
    INSTALL_COMMAND make install
    LOG_DOWNLOAD 1 LOG_UPDATE 1 LOG_CONFIGURE 1 LOG_BUILD 1 LOG_INSTALL 1
)

ExternalProject_Add(gcc-base
    PREFIX gcc-prefix
    STAMP_DIR gcc-prefix/src/gcc-stamp
    SOURCE_DIR gcc-prefix/src/gcc
    BINARY_DIR gcc-prefix/src/gcc-build
    DEPENDS mingw-w64-headers
    URL "ftp://ftp.gwdg.de/pub/misc/gcc/releases/gcc-5.4.0/gcc-5.4.0.tar.bz2"
    URL_HASH SHA512=2941cc950c8f2409a314df497631f9b0266211aa74746c1839c46e04f1c7c299afe2528d1ef16ea39def408a644ba48f97519ec7a7dd37d260c3e9423514265b
    CONFIGURE_COMMAND <SOURCE_DIR>/configure
        --target=${TARGET_ARCH}
        --prefix=${CMAKE_INSTALL_PREFIX}
        --libdir=${CMAKE_INSTALL_PREFIX}/lib
        --with-sysroot=${CMAKE_INSTALL_PREFIX}
        --disable-multilib
        --enable-languages=c,c++
        --disable-nls
        --disable-shared
        --disable-win32-registry
    BUILD_COMMAND make -j${MAKEJOBS} all-gcc
    INSTALL_COMMAND make install-gcc
    LOG_DOWNLOAD 1 LOG_CONFIGURE 1 LOG_BUILD 1 LOG_INSTALL 1
)

ExternalProject_Add(mingw-w64-crt
    PREFIX mingw-w64-prefix
    SOURCE_DIR mingw-w64-prefix/src
    DEPENDS gcc-base
    CONFIGURE_COMMAND ${EXEC} <SOURCE_DIR>/mingw-w64/mingw-w64-crt/configure
        --host=${TARGET_ARCH}
        --prefix=${MINGW_INSTALL_PREFIX}
        --with-sysroot=${CMAKE_INSTALL_PREFIX}
    BUILD_COMMAND ${MAKE}
    INSTALL_COMMAND ${MAKE} install
    LOG_DOWNLOAD 1 LOG_UPDATE 1 LOG_CONFIGURE 1 LOG_BUILD 1 LOG_INSTALL 1
)

ExternalProject_Add_Step(mingw-w64-headers autoreconf
    DEPENDEES download update patch
    DEPENDERS configure
    COMMAND ${EXEC} autoreconf -fi
    WORKING_DIRECTORY <SOURCE_DIR>/mingw-w64-crt
    LOG 1
)

ExternalProject_Add(gcc
    DEPENDS mingw-w64-crt
    SOURCE_DIR gcc-prefix/src
    CONFIGURE_COMMAND ""
    BUILD_COMMAND ${MAKE}
    INSTALL_COMMAND ${MAKE} install
    LOG_BUILD 1 LOG_INSTALL 1
)

ExternalProject_Add(winpthreads
    PREFIX mingw-w64-prefix
    SOURCE_DIR mingw-w64-prefix/src
    DEPENDS gcc
    CONFIGURE_COMMAND ${EXEC} <SOURCE_DIR>/mingw-w64/mingw-w64-libraries/winpthreads/configure
        --host=${TARGET_ARCH}
        --prefix=${MINGW_INSTALL_PREFIX}
        --disable-shared
        --enable-static
    BUILD_COMMAND ${MAKE}
    INSTALL_COMMAND ${MAKE} install
    LOG_CONFIGURE 1 LOG_BUILD 1 LOG_INSTALL 1
)
