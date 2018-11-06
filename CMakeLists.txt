cmake_minimum_required(VERSION 3.5)
set(CMAKE_CXX_STANDARD 11)
project(aws-lambda-runtime
    VERSION 0.1
    LANGUAGES CXX)

include(CheckCXXSourceCompiles)
include(CheckCXXCompilerFlag)

check_cxx_source_compiles("
#include <execinfo.h>

int main() {
    void *buffer[5];
    int nptrs = backtrace(buffer, 5);
    char **strings = backtrace_symbols(buffer, nptrs);

    return !!strings;
    }"  HAVE_BACKTRACE)


check_cxx_compiler_flag("-flto" LTO_CAPABLE)

add_library(${PROJECT_NAME}
    "src/logging.cpp"
    "src/runtime.cpp"
    )

target_include_directories(${PROJECT_NAME} PUBLIC
    $<BUILD_INTERFACE:${CMAKE_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>)

find_package(CURL REQUIRED)
target_link_libraries(${PROJECT_NAME} PRIVATE curl)

if(HAVE_BACKTRACE)
    target_compile_definitions(${PROJECT_NAME} PRIVATE -DHAVE_BACKTRACE)
endif()

target_compile_options(${PROJECT_NAME} PRIVATE
    "-fno-exceptions"
    "-fno-rtti"
    "-fvisibility=hidden"
    "-Wall"
    "-Wextra"
    "-Werror"
    "-Wconversion"
    "-Wno-sign-conversion")

#TODO: make this a CMake option/variable
if (LOG_VERBOSITY)
    target_compile_definitions(${PROJECT_NAME} PRIVATE "-DAWS_LAMBDA_LOG=${LOG_VERBOSITY}")
else ()
    target_compile_definitions(${PROJECT_NAME} PRIVATE "-DAWS_LAMBDA_LOG=0")
endif()

if (${LTO_CAPABLE})
    target_compile_options(${PROJECT_NAME} PRIVATE "-flto")
    target_link_libraries(${PROJECT_NAME} PRIVATE "-flto")
endif()

target_compile_features(${PROJECT_NAME} PRIVATE "cxx_std_11")

# installation
install(FILES
    "include/aws/lambda-runtime/runtime.h"
    DESTINATION "include/aws/lambda-runtime")

install(TARGETS ${PROJECT_NAME}
    EXPORT ${PROJECT_NAME}-targets
    ARCHIVE DESTINATION ${CMAKE_INSTALL_PREFIX}/lib
    LIBRARY DESTINATION ${CMAKE_INSTALL_PREFIX}/lib
    RUNTIME DESTINATION ${CMAKE_INSTALL_PREFIX}/bin)

configure_file("${CMAKE_SOURCE_DIR}/cmake/${PROJECT_NAME}-config.cmake"
    "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config.cmake"
  @ONLY)

export(EXPORT "${PROJECT_NAME}-targets" NAMESPACE AWS::)

install(EXPORT "${PROJECT_NAME}-targets"
    DESTINATION "${CMAKE_INSTALL_PREFIX}/lib/${PROJECT_NAME}/cmake/"
    NAMESPACE AWS::)

install(FILES "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config.cmake"
    DESTINATION "${CMAKE_INSTALL_PREFIX}/lib/${PROJECT_NAME}/cmake/")
