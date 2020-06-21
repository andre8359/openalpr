project(src)

set(CMAKE_BUILD_TYPE RelWithDebugInfo)
# set(CMAKE_BUILD_TYPE Debug)

include(${CMAKE_BINARY_DIR}/../conanbuildinfo.cmake)
conan_basic_setup()

cmake_minimum_required(VERSION 2.6)

# Set the OpenALPR version in cmake, and also add it as a DEFINE for the code to
# access
set(OPENALPR_MAJOR_VERSION "2")
set(OPENALPR_MINOR_VERSION "3")
set(OPENALPR_PATCH_VERSION "0")
set(OPENALPR_VERSION
    ${OPENALPR_MAJOR_VERSION}.${OPENALPR_MINOR_VERSION}.${OPENALPR_PATCH_VERSION}
)

add_definitions(-DOPENALPR_MAJOR_VERSION=${OPENALPR_MAJOR_VERSION})
add_definitions(-DOPENALPR_MINOR_VERSION=${OPENALPR_MINOR_VERSION})
add_definitions(-DOPENALPR_PATCH_VERSION=${OPENALPR_PATCH_VERSION})

set(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake_modules/")

add_definitions(-DINSTALL_PREFIX="${CMAKE_INSTALL_PREFIX}")

# TODO: switch to
# http://www.cmake.org/cmake/help/v2.8.5/cmake.html#module:GNUInstallDirs ?
if(NOT CMAKE_INSTALL_SYSCONFDIR)
    set(CMAKE_INSTALL_SYSCONFDIR "/etc")
endif()
if(NOT CMAKE_INSTALL_VARDIR)
    set(CMAKE_INSTALL_VARDIR "${CMAKE_INSTALL_PREFIX}/var")
endif()

if(NOT DEFINED WITH_DAEMON)
    set(WITH_DAEMON ON)
endif()

if(NOT DEFINED WITH_STATEDETECTION)
    set(WITH_STATEDETECTION OFF)
endif()

if(NOT DEFINED WITH_GPU_DETECTOR)
    set(WITH_GPU_DETECTOR OFF)
endif()

if(NOT DEFINED WITH_TESTS)
    set(WITH_TESTS ON)
endif()

if(NOT DEFINED WITH_BINDING_JAVA)
    set(WITH_BINDING_JAVA ON)
endif()

if(NOT DEFINED WITH_BINDING_PYTHON)
    set(WITH_BINDING_PYTHON ON)
endif()

if(NOT DEFINED WITH_BINDING_GO)
    set(WITH_BINDING_GO ON)
endif()

if(NOT DEFINED WITH_UTILITIES)
    set(WITH_UTILITIES ON)
endif()

if(WIN32 AND WITH_DAEMON)
    message(
        WARNING
            "Skipping alprd daemon installation, as it is not supported in Windows."
    )
    set(WITH_DAEMON OFF)
endif()

if(WIN32 AND WITH_TESTS)
    message(WARNING "Skipping unit tests, as it is not supported in Windows.")
    set(WITH_TESTS OFF)
endif()

include_directories(${CONAN_INCLUDE_DIRS_TESSERACT})
include_directories(${CONAN_INCLUDE_DIRS_OPENCV})

set(opencv_VERSION "3.4.3")
set(opencv_VERSION_MAJOR "3")
if(${opencv_VERSION} VERSION_LESS 2.4.7)
    message(FATAL_ERROR "OpenCV version is not compatible : ${opencv_VERSION}")
endif()

add_definitions(-DOPENCV_MAJOR_VERSION=${opencv_VERSION_MAJOR})

include_directories(./openalpr)

if(WIN32)
    add_definitions(-DWINDOWS)
    add_definitions(-DNOMINMAX)

    set(OPENALPR_LIB openalpr-static)

    # Extra linker dependencies for Windows
    set(Tesseract_LIBRARIES ${Tesseract_LIBRARIES} ws2_32.lib)
    set(Extra_LIBS ${CONAN_LIB_DIRS_ZLIB}/${CONAN_LIBS_ZLIB}.lib)
else()

    set(Extra_LIBS pthread)
    set(CMAKE_CSS_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -Wall ")
    set(OPENALPR_LIB openalpr)

endif()

if(NOT IOS)
    add_executable(alpr main.cpp)
endif()


if(WITH_STATEDETECTION)
    set(STATE_DETECTION_LIB statedetection)
    add_subdirectory(statedetection)
else()
    set(STATE_DETECTION_LIB "")
    add_definitions(-DSKIP_STATE_DETECTION=1)
endif()

if(NOT IOS)
    target_link_libraries(
        alpr
        ${OPENALPR_LIB}
        ${STATE_DETECTION_LIB}
        support
        video
        ${CONAN_LIBS}
        ${Extra_LIBS})
endif()

# Compile the alprd library on Unix-based OS
if(WITH_DAEMON)
    add_executable(alprd daemon.cpp daemon/daemonconfig.cpp daemon/beanstalk.c
                         daemon/beanstalk.cc)

    find_package(CURL REQUIRED)
    find_package(log4cplus REQUIRED)

    target_link_libraries(
        alprd
        ${OPENALPR_LIB}
        support
        video
        curl
        CONAN_PKG::opencv
        CONAN_PKG::tesseract
        ${log4cplus_LIBRARIES}
        ${Extra_LIBS})

endif()

if(WITH_UTILITIES)
    add_subdirectory(misc_utilities)
endif()

if(WITH_TESTS)
    add_subdirectory(tests)
endif()

add_subdirectory(openalpr)
add_subdirectory(video)

if(WITH_BINDING_JAVA)
    add_subdirectory(bindings/java)
endif()

if(WITH_BINDING_PYTHON)
    add_subdirectory(bindings/python)
endif()

if(WITH_BINDING_GO)
    set(OPENALPR_LIB_GO openalprgo)
    set(TAG_OPENALPR_LIB_GO "-l${OPENALPR_LIB_GO}")
    add_subdirectory(bindings/go)
endif()

if(NOT IOS)
    install(TARGETS alpr DESTINATION ${CMAKE_INSTALL_PREFIX}/bin)
    install(
        FILES ${CMAKE_SOURCE_DIR}/../doc/man/alpr.1
        DESTINATION ${CMAKE_INSTALL_PREFIX}/share/man/man1
        COMPONENT doc)
endif()
install(DIRECTORY ${CMAKE_SOURCE_DIR}/../runtime_data
        DESTINATION ${CMAKE_INSTALL_PREFIX}/share/openalpr)

# set runtime_data to reflect the current CMAKE_INSTALL_PREFIX
configure_file(${CMAKE_SOURCE_DIR}/../config/openalpr.conf.user
               ${CMAKE_CURRENT_BINARY_DIR}/config/openalpr.conf)
install(
    FILES ${CMAKE_CURRENT_BINARY_DIR}/config/openalpr.conf
    DESTINATION ${CMAKE_INSTALL_SYSCONFDIR}/openalpr/
    COMPONENT config)

configure_file(
    ${CMAKE_SOURCE_DIR}/../config/openalpr.conf.defaults
    ${CMAKE_CURRENT_BINARY_DIR}/share/openalpr/config/openalpr.defaults.conf)
install(
    FILES
        ${CMAKE_CURRENT_BINARY_DIR}/share/openalpr/config/openalpr.defaults.conf
    DESTINATION ${CMAKE_INSTALL_PREFIX}/share/openalpr/config
    COMPONENT config)

if(WITH_DAEMON)
    configure_file(${CMAKE_SOURCE_DIR}/../config/alprd.conf.user
                   ${CMAKE_CURRENT_BINARY_DIR}/config/alprd.conf)
    install(
        FILES ${CMAKE_CURRENT_BINARY_DIR}/config/alprd.conf
        DESTINATION ${CMAKE_INSTALL_SYSCONFDIR}/openalpr/
        COMPONENT config)

    configure_file(
        ${CMAKE_SOURCE_DIR}/../config/alprd.conf.defaults
        ${CMAKE_CURRENT_BINARY_DIR}/share/openalpr/config/alprd.defaults.conf)
    install(
        FILES
            ${CMAKE_CURRENT_BINARY_DIR}/share/openalpr/config/alprd.defaults.conf
        DESTINATION ${CMAKE_INSTALL_PREFIX}/share/openalpr/config
        COMPONENT config)

    install(TARGETS alprd DESTINATION ${CMAKE_INSTALL_PREFIX}/bin)
endif()

set(CPACK_PACKAGE_VERSION ${OPENALPR_VERSION})
set(CPACK_SET_DESTDIR "on")
set(CPACK_GENERATOR "DEB;TGZ")
set(CPACK_PACKAGE_NAME "openalpr")
set(CPACK_INSTALL_CMAKE_PROJECTS "${CMAKE_SOURCE_DIR}/build;src;ALL;/")
set(CPACK_CMAKE_GENERATOR "Unix Makefiles")

set(CPACK_STRIP_FILES "1")
set(CPACK_DEBIAN_PACKAGE_PRIORITY "optional")
set(CPACK_DEBIAN_PACKAGE_SECTION "video")
set(CPACK_DEBIAN_ARCHITECTURE ${CMAKE_SYSTEM_PROCESSOR})
set(CPACK_DEBIAN_PACKAGE_DEPENDS
    "libc6 (>= 2.1.3), libgcc1 (>= 4.1.1), libtesseract3 (>= 3.0.3), libopencv-core2.4 (>= 2.4.8), libopencv-objdetect2.4 (>= 2.4.8), libopencv-highgui2.4 (>= 2.4.8), libopencv-imgproc2.4 (>= 2.4.8), libopencv-flann2.4 (>= 2.4.8), libopencv-features2d2.4 (>= 2.4.8), libopencv-video2.4 (>= 2.4.8), libopencv-gpu2.4 (>=2.4.8), liblog4cplus-1.0-4, libcurl3, beanstalkd"
)

set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_SOURCE_DIR}/../LICENSE")
set(CPACK_PACKAGE_DESCRIPTION
    "OpenALPR - Open Source Automatic License Plate Recognition")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY
    "OpenALPR is an open source Automatic License Plate Recognition library written in C++.  The library analyzes images and identifies license plates. The output is the text representation of any license plate characters found in the image.  Check out a live online demo here: http://www.openalpr.com/demo.html"
)
set(CPACK_PACKAGE_CONTACT "Matt Hill <matt@ndu.com>")
set(CPACK_PACKAGE_FILE_NAME
    "${CPACK_PACKAGE_NAME}_${CPACK_PACKAGE_VERSION}_${CPACK_DEBIAN_ARCHITECTURE}"
)

set(CPACK_COMPONENTS_ALL Libraries ApplicationData)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
set(CMAKE_CXX_STANDARD 11)
cmake_minimum_required(VERSION 3.1)
# Enable C++11
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED TRUE)
include(CPack)

# ----------------------------------------------------------------------------
# Uninstall target, for "make uninstall"
# http://www.cmake.org/Wiki/CMake_FAQ#Can_I_do_.22make_uninstall.22_with_CMake.3F
# ----------------------------------------------------------------------------
configure_file("${CMAKE_MODULE_PATH}/templates/cmake_uninstall.cmake.in"
               "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake" @ONLY)

add_custom_target(
    uninstall COMMAND ${CMAKE_COMMAND} -P
                      "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake")
