project(XS)

# Find the Perl interpreter, add local-lib to PATH and PERL5LIB environment variables,
# so the locally installed modules (mainly the Alien::wxPerl) will be reached.
if (WIN32)
    set(ENV_PATH_SEPARATOR ";")
else()
    set(ENV_PATH_SEPARATOR ":")
endif()

# Install the XS.pm and XS.{so,dll,bundle} into the local-lib directory.
set(PERL_LOCAL_LIB_DIR ${PROJECT_SOURCE_DIR}/../local-lib)

set(ENV{PATH}     "${PERL_LOCAL_LIB_DIR}/bin${ENV_PATH_SEPARATOR}$ENV{PATH}")
set(PERL_INCLUDE  "${PERL_LOCAL_LIB_DIR}/lib/perl5${ENV_PATH_SEPARATOR}$ENV{PERL5LIB}")
message("PATH: $ENV{PATH}")
message("PERL_INCLUDE: ${PERL_INCLUDE}")
find_package(Perl REQUIRED)
if (WIN32)
    # On Windows passing the PERL5LIB variable causes various problems (such as with MAX_PATH and others),
    # basically I've found no good way to do it on Windows.
    set(PERL5LIB_ENV_CMD "")
else()
    set(PERL5LIB_ENV_CMD ${CMAKE_COMMAND} -E env PERL5LIB=${PERL_INCLUDE})
endif()

# Perl specific stuff
find_package(PerlLibs REQUIRED)
set(PerlEmbed_DEBUG 1)
find_package(PerlEmbed REQUIRED)

# Generate the Slic3r Perl module (XS) typemap file.
set(MyTypemap ${CMAKE_CURRENT_BINARY_DIR}/typemap)
add_custom_command(
        OUTPUT ${MyTypemap}
        DEPENDS ${CMAKE_CURRENT_LIST_DIR}/xsp/my.map
        COMMAND ${PERL5LIB_ENV_CMD} ${PERL_EXECUTABLE} -MExtUtils::Typemaps -MExtUtils::Typemaps::Basic -e "$typemap = ExtUtils::Typemaps->new(file => \"${CMAKE_CURRENT_LIST_DIR}/xsp/my.map\"); $typemap->merge(typemap => ExtUtils::Typemaps::Basic->new); $typemap->write(file => \"${MyTypemap}\")"
        VERBATIM
)

# Generate the Slic3r Perl module (XS) main.xs file.
set(XS_MAIN_XS ${CMAKE_CURRENT_BINARY_DIR}/main.xs)
set(XSP_DIR ${CMAKE_CURRENT_SOURCE_DIR}/xsp)
#FIXME list the dependecies explicitely, add dependency on the typemap.
set(XS_XSP_FILES
    ${XSP_DIR}/Config.xsp
    ${XSP_DIR}/ExPolygon.xsp
    ${XSP_DIR}/Geometry.xsp
    ${XSP_DIR}/Line.xsp
    ${XSP_DIR}/Model.xsp
    ${XSP_DIR}/Point.xsp
    ${XSP_DIR}/Polygon.xsp
    ${XSP_DIR}/Polyline.xsp
    ${XSP_DIR}/Print.xsp
    ${XSP_DIR}/TriangleMesh.xsp
    ${XSP_DIR}/XS.xsp
)
foreach (file ${XS_XSP_FILES})
    if (MSVC)
        # Visual Studio C compiler has issues with FILE pragmas containing quotes.
        set(INCLUDE_COMMANDS "${INCLUDE_COMMANDS}INCLUDE_COMMAND: $^X -MExtUtils::XSpp::Cmd -e xspp -- -t ${CMAKE_CURRENT_LIST_DIR}/xsp/typemap.xspt ${file}\n")
    else ()
        set(INCLUDE_COMMANDS "${INCLUDE_COMMANDS}INCLUDE_COMMAND: $^X -MExtUtils::XSpp::Cmd -e xspp -- -t \"${CMAKE_CURRENT_LIST_DIR}/xsp/typemap.xspt\" \"${file}\"\n")
    endif ()
endforeach ()
configure_file(main.xs.in ${XS_MAIN_XS} @ONLY) # Insert INCLUDE_COMMANDS into main.xs

# Generate the Slic3r Perl module (XS) XS.cpp file.
#FIXME add the dependency on main.xs and typemap.
set(XS_MAIN_CPP ${CMAKE_CURRENT_BINARY_DIR}/XS.cpp)
add_custom_command(
        OUTPUT ${XS_MAIN_CPP}
        DEPENDS ${MyTypemap} ${XS_XSP_FILES} ${CMAKE_CURRENT_LIST_DIR}/xsp/typemap.xspt
        COMMAND ${PERL5LIB_ENV_CMD} xsubpp -typemap typemap -output ${XS_MAIN_CPP} -hiertype ${XS_MAIN_XS}
        VERBATIM
)

# Define the Perl XS shared library.
if(APPLE)
    set(XS_SHARED_LIBRARY_TYPE MODULE)
else()
    set(XS_SHARED_LIBRARY_TYPE SHARED)
endif()
add_library(XS ${XS_SHARED_LIBRARY_TYPE}
    ${XS_MAIN_CPP}
    src/perlglue.cpp
    src/ppport.h
    src/xsinit.h
    xsp/my.map
    # mytype.map is empty. Is it required by Build.PL or the Perl xspp module?
    xsp/mytype.map
    # Used by Perl xsubpp to generate XS.cpp
    xsp/typemap.xspt
)
if(APPLE)
    set_target_properties(XS PROPERTIES BUNDLE TRUE)
    # Ignore undefined symbols of the perl interpreter, they will be found in the caller image.
    target_link_libraries(XS "-undefined dynamic_lookup")
endif()
target_link_libraries(XS libslic3r)

target_include_directories(XS PRIVATE src ${LIBDIR}/libslic3r)
target_compile_definitions(XS PRIVATE -DSLIC3RXS)
set_target_properties(XS PROPERTIES PREFIX "") # Prevent cmake from generating libXS.so instead of XS.so

if (APPLE)
    # -liconv: boost links to libiconv by default
    target_link_libraries(XS "-liconv -framework IOKit" "-framework CoreFoundation" -lc++)
elseif (MSVC)
    target_link_libraries(XS )
else ()
    target_link_libraries(XS -lstdc++)
endif ()

# Windows specific stuff
if (WIN32)
    target_compile_definitions(XS PRIVATE -DNOGDI -DNOMINMAX -DHAS_BOOL)
endif ()

# SLIC3R_MSVC_PDB
if (MSVC AND SLIC3R_MSVC_PDB AND "${CMAKE_BUILD_TYPE}" STREQUAL "Release")
    set_target_properties(XS PROPERTIES
        COMPILE_FLAGS "/Zi"
        LINK_FLAGS "/DEBUG /OPT:REF /OPT:ICF"
    )
endif()

if (CMAKE_BUILD_TYPE MATCHES DEBUG)
    target_compile_definitions(XS PRIVATE -DSLIC3R_DEBUG -DDEBUG -D_DEBUG)
else ()
    target_compile_definitions(XS PRIVATE -DNDEBUG)
endif ()

target_include_directories(XS PRIVATE ${PERL_INCLUDE_PATH})
target_compile_options(XS PRIVATE ${PerlEmbed_CCFLAGS})

if (WIN32)
    target_link_libraries(XS ${PERL_LIBRARY})
endif()


set(PERL_LOCAL_LIB_ARCH_DIR "${PERL_LOCAL_LIB_DIR}/lib/perl5/${PerlEmbed_ARCHNAME}")
add_custom_command(
    TARGET XS
    POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E make_directory "${PERL_LOCAL_LIB_ARCH_DIR}/auto/Slic3r/XS/"
        COMMAND ${CMAKE_COMMAND} -E copy "$<TARGET_FILE:XS>" "${PERL_LOCAL_LIB_ARCH_DIR}/auto/Slic3r/XS/"
        COMMAND ${CMAKE_COMMAND} -E make_directory "${PERL_LOCAL_LIB_ARCH_DIR}/Slic3r/"
        COMMAND ${CMAKE_COMMAND} -E copy "${CMAKE_CURRENT_SOURCE_DIR}/lib/Slic3r/XS.pm" "${PERL_LOCAL_LIB_ARCH_DIR}/Slic3r/"
    COMMENT "Installing XS.pm and XS.{so,dll,bundle} into the local-lib directory ..."
)
if(APPLE)
    add_custom_command(
        TARGET XS
        POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E rename "${PERL_LOCAL_LIB_ARCH_DIR}/auto/Slic3r/XS/XS" "${PERL_LOCAL_LIB_ARCH_DIR}/auto/Slic3r/XS/XS.bundle"
    )
endif()

if (MSVC)
    # Here we associate some additional properties with the MSVC project to enable compilation and debugging out of the box.
    get_filename_component(PROPS_PERL_BIN_PATH "${PERL_EXECUTABLE}" DIRECTORY)
    string(REPLACE "/" "\\" PROPS_PERL_BIN_PATH "${PROPS_PERL_BIN_PATH}")
    string(REPLACE "/" "\\" PROPS_PERL_EXECUTABLE "${PERL_EXECUTABLE}")
    string(REPLACE "/" "\\" PROPS_CMAKE_SOURCE_DIR "${CMAKE_SOURCE_DIR}")
    configure_file("../cmake/msvc/xs.wperl.props.in" "${CMAKE_BINARY_DIR}/xs.wperl.props" NEWLINE_STYLE CRLF)
    set_target_properties(XS PROPERTIES VS_USER_PROPS "${CMAKE_BINARY_DIR}/xs.wperl.props")

    if ("${CMAKE_SIZEOF_VOID_P}" STREQUAL "8")
        set(_bits 64)
    elseif ("${CMAKE_SIZEOF_VOID_P}" STREQUAL "4")
        set(_bits 32)
    endif ()
    add_custom_command(TARGET XS POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy ${TOP_LEVEL_PROJECT_DIR}/deps/+GMP/gmp/lib/win${_bits}/libgmp-10.dll "${PERL_LOCAL_LIB_ARCH_DIR}/auto/Slic3r/XS/"
        COMMENT "Installing gmp runtime into the local-lib directory ..."
        VERBATIM)

    add_custom_command(TARGET XS POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy ${TOP_LEVEL_PROJECT_DIR}/deps/+MPFR/mpfr/lib/win${_bits}/libmpfr-4.dll "${PERL_LOCAL_LIB_ARCH_DIR}/auto/Slic3r/XS/"
        COMMENT "Installing mpfr runtime into the local-lib directory ..."
        VERBATIM)
endif()

# Installation
install(TARGETS XS DESTINATION ${PERL_VENDORARCH}/auto/Slic3r/XS)
install(FILES lib/Slic3r/XS.pm DESTINATION ${PERL_VENDORLIB}/Slic3r)

# Unit / integration tests
enable_testing()
get_filename_component(PERL_BIN_PATH "${PERL_EXECUTABLE}" DIRECTORY)
if (MSVC)
    set(PERL_PROVE "${PERL_BIN_PATH}/prove.bat")
else ()
    set(PERL_PROVE "${PERL_BIN_PATH}/prove")
endif ()

set(PERL_ENV_VARS "")
if (CMAKE_SYSTEM_NAME STREQUAL "Linux" AND NOT CMAKE_CROSSCOMPILING AND ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU" OR "${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang"))
    if (SLIC3R_ASAN OR SLIC3R_UBSAN)
        set(PERL_ENV_VARS env)
    endif ()

    if (SLIC3R_ASAN)
        # Find the location of libasan.so for passing it into LD_PRELOAD. It works with GCC and Clang on Linux.
        # On Centos 7 calling "gcc -print-file-name=libasan.so" returns path to "ld script" instead of path to shared library.
        set(_asan_compiled_bin ${CMAKE_CURRENT_BINARY_DIR}/detect_libasan)
        set(_asan_source_file ${_asan_compiled_bin}.c)
        # Compile and link simple C application with enabled address sanitizer.
        file(WRITE ${_asan_source_file} "int main(){}")
        include(GetPrerequisites)
        execute_process(COMMAND ${CMAKE_C_COMPILER} ${_asan_source_file} -fsanitize=address -lasan -o ${_asan_compiled_bin})
        # Extract from the compiled application absolute path of libasan.
        get_prerequisites(${_asan_compiled_bin} _asan_shared_libraries_list 0 0 "" "")
        list(FILTER _asan_shared_libraries_list INCLUDE REGEX libasan)
        set(PERL_ENV_VARS ${PERL_ENV_VARS} "LD_PRELOAD=${_asan_shared_libraries_list}")

        # Suppressed memory leak reports that come from Perl.
        set(PERL_LEAK_SUPPRESSION_FILE ${CMAKE_CURRENT_BINARY_DIR}/leak_suppression.txt)
        file(WRITE ${PERL_LEAK_SUPPRESSION_FILE}
                "leak:Perl_safesysmalloc\n"
                "leak:Perl_safesyscalloc\n"
                "leak:Perl_safesysrealloc\n"
                "leak:__newlocale\n")

        # Suppress a few memory leak reports and disable informing about suppressions.
        # Print reports about memory leaks but exit with zero exit code when any memory leaks is found to make unit tests pass.
        set(PERL_ENV_VARS ${PERL_ENV_VARS} "LSAN_OPTIONS=suppressions=${PERL_LEAK_SUPPRESSION_FILE}:print_suppressions=0:exitcode=0")
    endif ()

    if (SLIC3R_UBSAN)
        # Do not show full stacktrace for reports from UndefinedBehaviorSanitizer in Perl tests.
        set(PERL_ENV_VARS ${PERL_ENV_VARS} "UBSAN_OPTIONS=print_stacktrace=0")
    endif ()
endif ()

add_test (NAME xs COMMAND ${PERL_ENV_VARS} "${PERL_EXECUTABLE}" ${PERL_PROVE} -I ${PERL_LOCAL_LIB_DIR}/lib/perl5 WORKING_DIRECTORY ${PROJECT_SOURCE_DIR})
add_test (NAME integration COMMAND ${PERL_ENV_VARS} "${PERL_EXECUTABLE}" ${PERL_PROVE} WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}/..)
