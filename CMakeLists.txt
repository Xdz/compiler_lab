cmake_minimum_required(VERSION 3.30)

project(compiler-lab)

set(CMAKE_CXX_STANDARD_DEFAULT 20)

add_compile_options(-Wall -Werror -O2)

include_directories(compiler-backend/include)

add_executable(compiler_backend)

target_link_libraries()