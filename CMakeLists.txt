cmake_minimum_required( VERSION 3.3.2 )

project(exiv2          # use TWEAK to categorize the build
    VERSION 0.27.4     # 0.27.4    = GM               (tagged and released)
                       # 0.27.4.9  = 27.4.9 Development
                       # 0.27.4.00 = GM Preview
                       # 0.27.4.3  = RC3              (tagged and released)
                       # 0.27.4.30 = RC2 Preview
                       # 0.27.4.39 = RC2 Development
    LANGUAGES CXX
)
include(cmake/mainSetup.cmake  REQUIRED)

# options and their default values
option( BUILD_SHARED_LIBS             "Build exiv2lib as a shared library"                    ON  )
option( EXIV2_ENABLE_XMP              "Build with XMP metadata support"                       ON  )
option( EXIV2_ENABLE_EXTERNAL_XMP     "Use external version of XMP"                           OFF )
option( EXIV2_ENABLE_PNG              "Build with png support (requires libz)"                ON  )
option( EXIV2_ENABLE_NLS              "Build native language support (requires gettext)"      OFF )
option( EXIV2_ENABLE_PRINTUCS2        "Build with Printucs2"                                  ON  )
option( EXIV2_ENABLE_LENSDATA         "Build including lens data"                             ON  )
option( EXIV2_ENABLE_VIDEO            "Build video support into library"                      OFF )
option( EXIV2_ENABLE_DYNAMIC_RUNTIME  "Use dynamic runtime (used for static libs)"            ON  )
option( EXIV2_ENABLE_WIN_UNICODE      "Use Unicode paths (wstring) on Windows"                OFF )
option( EXIV2_ENABLE_WEBREADY         "Build webready support into library"                   OFF )
option( EXIV2_ENABLE_CURL             "USE Libcurl for HttpIo (WEBREADY)"                     OFF )
option( EXIV2_ENABLE_SSH              "USE Libssh for SshIo (WEBREADY)"                       OFF )
option( EXIV2_ENABLE_BMFF             "Build with BMFF support"                               OFF )

option( EXIV2_BUILD_SAMPLES           "Build sample applications"                             ON  )
option( EXIV2_BUILD_EXIV2_COMMAND     "Build exiv2 command-line executable"                   ON  )
option( EXIV2_BUILD_UNIT_TESTS        "Build unit tests"                                      OFF )
option( EXIV2_BUILD_DOC               "Add 'doc' target to generate documentation"            OFF )

# Only intended to be used by Exiv2 developers/contributors
option( EXIV2_TEAM_EXTRA_WARNINGS     "Add more sanity checks using compiler flags"           OFF )
option( EXIV2_TEAM_WARNINGS_AS_ERRORS "Treat warnings as errors"                              OFF )
option( EXIV2_TEAM_USE_SANITIZERS     "Enable ASAN and UBSAN when available"                  OFF )
option( EXIV2_TEAM_PACKAGING          "Additional stuff for generating packages"              OFF )
set(EXTRA_COMPILE_FLAGS " ")

mark_as_advanced(
    EXIV2_TEAM_EXTRA_WARNINGS
    EXIV2_TEAM_WARNINGS_AS_ERRORS
    EXIV2_ENABLE_EXTERNAL_XMP
    EXTRA_COMPILE_FLAGS
    EXIV2_TEAM_USE_SANITIZERS
)

option( BUILD_WITH_CCACHE             "Use ccache to speed up compilations"                   OFF )
option( BUILD_WITH_COVERAGE           "Add compiler flags to generate coverage stats"         OFF )

set( PACKAGE_BUGREPORT                "http://github.com/exiv2/exiv2" )
set( PACKAGE_URL                      "https://exiv2.org")
set( PROJECT_DESCRIPTION              "Exif/IPTC/Xmp C++ metadata library and tools plus ICC Profiles, Previews and more.")

if ( EXIV2_ENABLE_EXTERNAL_XMP )
    set(EXIV2_ENABLE_XMP OFF)
endif()

if( EXIV2_BUILD_UNIT_TESTS )
    set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON) # Requires CMake 3.3.3
endif()

include(cmake/findDependencies.cmake   REQUIRED)
include(cmake/compilerFlags.cmake      REQUIRED)
include(cmake/generateConfigFile.cmake REQUIRED)

if (EXIV2_BUILD_DOC)
    include(cmake/generateDoc.cmake        REQUIRED)
    generate_documentation("${PROJECT_SOURCE_DIR}/cmake/Doxyfile.in")
endif()


include_directories(${CMAKE_BINARY_DIR})  # Make the exv_conf.h file visible for the full project

if( EXIV2_ENABLE_XMP )
    add_subdirectory( xmpsdk )
endif()

include(cmake/compilerFlagsExiv2.cmake REQUIRED)

add_subdirectory( include )
add_subdirectory( src )

if( EXIV2_BUILD_UNIT_TESTS )
    add_subdirectory ( unitTests )
    add_custom_target(unit_test
        COMMAND env EXIV2_BINDIR="${CMAKE_RUNTIME_OUTPUT_DIRECTORY}" python3 runner.py bash_tests/unit_test.py
        WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/tests"
    )
endif()

add_custom_target(version_test
    COMMAND env EXIV2_BINDIR="${CMAKE_RUNTIME_OUTPUT_DIRECTORY}" python3 runner.py bash_tests/version_test.py
    WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/tests"
)

if( EXIV2_BUILD_SAMPLES )
    ##
    # tests
    if( EXIV2_BUILD_UNIT_TESTS )
        add_custom_target(tests
            COMMAND env EXIV2_BINDIR="${CMAKE_RUNTIME_OUTPUT_DIRECTORY}" python3 runner.py bash_tests/unit_test.py
            COMMAND env EXIV2_BINDIR="${CMAKE_RUNTIME_OUTPUT_DIRECTORY}" python3 runner.py bash_tests/testcases.py --verbose
            COMMAND env EXIV2_BINDIR="${CMAKE_RUNTIME_OUTPUT_DIRECTORY}" python3 runner.py 
            COMMAND env EXIV2_BINDIR="${CMAKE_RUNTIME_OUTPUT_DIRECTORY}" python3 runner.py bash_tests/version_test.py
            WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/tests"
        )
    else()
        add_custom_target(tests
            COMMAND env EXIV2_BINDIR="${CMAKE_RUNTIME_OUTPUT_DIRECTORY}" python3 runner.py bash_tests/testcases.py --verbose
            COMMAND env EXIV2_BINDIR="${CMAKE_RUNTIME_OUTPUT_DIRECTORY}" python3 runner.py
            COMMAND env EXIV2_BINDIR="${CMAKE_RUNTIME_OUTPUT_DIRECTORY}" python3 runner.py bash_tests/version_test.py
            WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/tests"
        )
    endif()
    add_custom_target(python_tests
        COMMAND env EXIV2_BINDIR="${CMAKE_RUNTIME_OUTPUT_DIRECTORY}" python3 runner.py
        WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/tests"
    )
    add_custom_target(bash_tests
        COMMAND env EXIV2_BINDIR="${CMAKE_RUNTIME_OUTPUT_DIRECTORY}" python3 runner.py bash_tests/testcases.py --verbose
        WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/tests"
    )
    add_subdirectory( samples )
    get_directory_property(SAMPLES DIRECTORY samples DEFINITION APPLICATIONS)
    add_dependencies(tests exiv2lib exiv2 ${SAMPLES})
endif()

if( EXIV2_ENABLE_NLS )
    add_subdirectory( po )
endif()

if (EXIV2_TEAM_PACKAGING)
    include(cmake/packaging.cmake)
endif()

join_paths(libdir_for_pc_file "\${prefix}" "${CMAKE_INSTALL_LIBDIR}")
join_paths(includedir_for_pc_file "\${prefix}" "${CMAKE_INSTALL_INCLUDEDIR}")
configure_file(cmake/exiv2.pc.in exiv2.pc @ONLY)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/exiv2.pc DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)

# ******************************************************************************
# Man page
install( FILES ${PROJECT_SOURCE_DIR}/man/man1/exiv2.1 DESTINATION ${CMAKE_INSTALL_MANDIR}/man1 )


include(cmake/printSummary.cmake)

# That's all Folks!
##
