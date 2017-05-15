cmake_minimum_required(VERSION 3.0)
set(CMAKE_ALLOW_LOOSE_LOOP_CONSTRUCTS ON)
include ("from_hex.cmake")
set(OPENSSL_INCLUDE_DIR ${NODEJS_ROOT}/include)
if (OPENSSL_INCLUDE_DIR AND EXISTS "${OPENSSL_INCLUDE_DIR}/openssl/opensslv.h")
    file(STRINGS "${OPENSSL_INCLUDE_DIR}/openssl/opensslv.h" openssl_version_str
            REGEX "^#[\t ]*define[\t ]+OPENSSL_VERSION_NUMBER[\t ]+0x([0-9a-fA-F])+.*")

    if (openssl_version_str)
        # The version number is encoded as 0xMNNFFPPS: major minor fix patch status
        # The status gives if this is a developer or prerelease and is ignored here.
        # Major, minor, and fix directly translate into the version numbers shown in
        # the string. The patch field translates to the single character suffix that
        # indicates the bug fix state, which 00 -> nothing, 01 -> a, 02 -> b and so
        # on.

        string(REGEX REPLACE "^.*OPENSSL_VERSION_NUMBER[\t ]+0x([0-9a-fA-F])([0-9a-fA-F][0-9a-fA-F])([0-9a-fA-F][0-9a-fA-F])([0-9a-fA-F][0-9a-fA-F])([0-9a-fA-F]).*$"
                "\\1;\\2;\\3;\\4;\\5" OPENSSL_VERSION_LIST "${openssl_version_str}")
        list(GET OPENSSL_VERSION_LIST 0 OPENSSL_VERSION_MAJOR)
        list(GET OPENSSL_VERSION_LIST 1 OPENSSL_VERSION_MINOR)
        from_hex("${OPENSSL_VERSION_MINOR}" OPENSSL_VERSION_MINOR)
        list(GET OPENSSL_VERSION_LIST 2 OPENSSL_VERSION_FIX)
        from_hex("${OPENSSL_VERSION_FIX}" OPENSSL_VERSION_FIX)
        list(GET OPENSSL_VERSION_LIST 3 OPENSSL_VERSION_PATCH)

        if (NOT OPENSSL_VERSION_PATCH STREQUAL "00")
            from_hex("${OPENSSL_VERSION_PATCH}" _tmp)
            # 96 is the ASCII code of 'a' minus 1
            math(EXPR OPENSSL_VERSION_PATCH_ASCII "${_tmp} + 96")
            unset(_tmp)
            # Once anyone knows how OpenSSL would call the patch versions beyond 'z'
            # this should be updated to handle that, too. This has not happened yet
            # so it is simply ignored here for now.
            string(ASCII "${OPENSSL_VERSION_PATCH_ASCII}" OPENSSL_VERSION_PATCH_STRING)
        endif ()

        set(OPENSSL_VERSION_FOR_PROJECT "${OPENSSL_VERSION_MAJOR}.${OPENSSL_VERSION_MINOR}.${OPENSSL_VERSION_FIX}")
        set(OPENSSL_VERSION "${OPENSSL_VERSION_MAJOR}.${OPENSSL_VERSION_MINOR}.${OPENSSL_VERSION_FIX}${OPENSSL_VERSION_PATCH_STRING}")
    endif ()
else ()
    message(FATAL_ERROR "invalid NODEJS_ROOT : ${NODEJS_ROOT}")
endif ()

project(OpenSSL VERSION "${OPENSSL_VERSION_FOR_PROJECT}" LANGUAGES C)

set(VERSION "${PROJECT_VERSION}")

set(INSTALL_BIN_DIR "${CMAKE_INSTALL_PREFIX}/bin" CACHE PATH "Installation directory for executables")
set(INSTALL_LIB_DIR "${CMAKE_INSTALL_PREFIX}/lib" CACHE PATH "Installation directory for libraries")
set(INSTALL_INC_DIR "${CMAKE_INSTALL_PREFIX}/include" CACHE PATH "Installation directory for headers")
set(INSTALL_MAN_DIR "${CMAKE_INSTALL_PREFIX}/share/man" CACHE PATH "Installation directory for manual pages")
set(INSTALL_PKGCONFIG_DIR "${CMAKE_INSTALL_PREFIX}/share/pkgconfig" CACHE PATH "Installation directory for pkgconfig (.pc) files")


if (NOT SKIP_INSTALL_LIBRARIES AND NOT SKIP_INSTALL_ALL)
    ####
    # Installation (https://github.com/forexample/package-example)

    set(config_install_dir "lib/cmake/${PROJECT_NAME}")
    set(include_install_dir "include")

    set(generated_dir "${CMAKE_CURRENT_BINARY_DIR}/generated")

    set(version_config "${generated_dir}/${PROJECT_NAME}ConfigVersion.cmake")
    set(project_config "${generated_dir}/${PROJECT_NAME}Config.cmake")
    set(targets_export_name "${PROJECT_NAME}Targets")
    set(namespace "${PROJECT_NAME}::")

    include(CMakePackageConfigHelpers)

    # Note: PROJECT_VERSION is used as a VERSION
    write_basic_package_version_file(
            "${version_config}" COMPATIBILITY SameMajorVersion
    )

    # Note: variable 'targets_export_name' used
    configure_file("cmake/Config.cmake.in" "${project_config}" @ONLY)

    install(
            FILES "${project_config}" "${version_config}"
            DESTINATION "${config_install_dir}"
    )

endif ()
