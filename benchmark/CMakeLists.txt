cmake_minimum_required(VERSION 3.6)

set(CMAKE_BUILD_TYPE RelWithDebInfo CACHE STRING "build type")

project(srpc_benchmark
		LANGUAGES C CXX
)

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR})

if (NOT "$ENV{LIBRARY_PATH}" STREQUAL "")
	string(REPLACE ":" ";" LIBRARY_PATH $ENV{LIBRARY_PATH})
	set(CMAKE_SYSTEM_LIBRARY_PATH ${LIBRARY_PATH};${CMAKE_SYSTEM_LIBRARY_PATH})
endif ()

if (NOT "$ENV{CPLUS_INCLUDE_PATH}" STREQUAL "")
	string(REPLACE ":" ";" INCLUDE_PATH $ENV{CPLUS_INCLUDE_PATH})
	set(CMAKE_SYSTEM_INCLUDE_PATH ${INCLUDE_PATH};${CMAKE_SYSTEM_INCLUDE_PATH})
endif ()

find_package(OpenSSL REQUIRED)

set(protobuf_MODULE_COMPATIBLE ON CACHE BOOL "")
if (WIN32)
	find_package(Protobuf CONFIG REQUIRED)
else ()
	find_package(Protobuf REQUIRED)
endif ()

find_package(Workflow REQUIRED CONFIG HINTS ../workflow)

#if (APPLE)
#	if (WORKFLOW_INCLUDE_DIR)
#		set (WORKFLOW_INCLUDE_DIR ${Workflow_DIR}/_include)
#		set (WORKFLOW_LIB_DIR ${Workflow_DIR}/_lib)
#	else ()
#		find_package(Workflow REQUIRED)
#	endif ()
#else ()
#	find_package(Workflow REQUIRED)
#endif()

find_package(srpc REQUIRED CONFIG HINTS ..)
include_directories(
	${OPENSSL_INCLUDE_DIR}
	${CMAKE_CURRENT_BINARY_DIR}
	${Protobuf_INCLUDE_DIR}
	${WORKFLOW_INCLUDE_DIR}
	${SRPC_INCLUDE_DIR}
)

if (WIN32)
	link_directories(${SRPC_LIB_DIR} ${WORKFLOW_LIB_DIR} ${Protobuf_LIB_DIR})
	set(SRPC_GEN_PROGRAM ${SRPC_BIN_DIR}/Debug/srpc_generator.exe)
else ()
	get_filename_component(Protobuf_LIB_DIR ${Protobuf_LIBRARY} DIRECTORY)
	link_directories(${SRPC_LIB_DIR} ${WORKFLOW_LIB_DIR} ${Protobuf_LIB_DIR})
	set(SRPC_GEN_PROGRAM ${SRPC_BIN_DIR}/srpc_generator)
endif ()

protobuf_generate_cpp(PROTO_SRCS PROTO_HDRS benchmark_pb.proto)

add_custom_target(
	BENCHMARK_GEN ALL
	COMMAND ${SRPC_GEN_PROGRAM} protobuf ${PROJECT_SOURCE_DIR}/benchmark_pb.proto  ${PROJECT_SOURCE_DIR}
	COMMAND ${SRPC_GEN_PROGRAM} thrift   ${PROJECT_SOURCE_DIR}/benchmark_thrift.thrift ${PROJECT_SOURCE_DIR}
	COMMENT "sprc generator..."
)

if (WIN32)
	set(CMAKE_C_FLAGS   "${CMAKE_C_FLAGS}   /MP /wd4200")
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /MP /wd4200 /Zc:__cplusplus /std:c++14")
else ()
	set(CMAKE_C_FLAGS   "${CMAKE_C_FLAGS}   -Wall -fPIC -pipe -std=gnu90")
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -fPIC -pipe -std=c++11 -fno-exceptions")
endif ()

if (APPLE)
	set(SRPC_LIB srpc workflow pthread OpenSSL::SSL OpenSSL::Crypto protobuf z)
elseif (WIN32)
	set(SRPC_LIB srpc workflow ws2_32 wsock32 OpenSSL::SSL OpenSSL::Crypto)
else ()
	set(SRPC_LIB srpc)
endif ()

add_executable(server server.cc ${PROTO_SRCS} ${PROTO_HDRS})
target_link_libraries(server ${SRPC_LIB})
add_dependencies(server BENCHMARK_GEN)

add_executable(client client.cc ${PROTO_SRCS} ${PROTO_HDRS})
target_link_libraries(client ${SRPC_LIB})
add_dependencies(client BENCHMARK_GEN)

add_executable(client_cdf client_cdf.cc ${PROTO_SRCS} ${PROTO_HDRS})
target_link_libraries(client_cdf ${SRPC_LIB})
add_dependencies(client_cdf BENCHMARK_GEN)

add_executable(proxy proxy.cc ${PROTO_SRCS} ${PROTO_HDRS})
target_link_libraries(proxy ${SRPC_LIB})
add_dependencies(proxy BENCHMARK_GEN)

