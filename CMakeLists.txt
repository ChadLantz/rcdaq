cmake_minimum_required(VERSION 3.25)
project(RCDAQ VERSION 1.0 LANGUAGES C CXX)

include(GNUInstallDirs)
include(CheckIncludeFile)

option(WRITEPRDF "Compile for legacy PRDF version" OFF)

set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

find_package(Threads REQUIRED)

check_include_file("rpc/rpc.h" HAVE_RPC_RPC_H)
set(RCDAQ_RPC_INCLUDE_DIR "")
set(RCDAQ_RPC_LINK_LIBS "")
if(NOT HAVE_RPC_RPC_H)
	set(RCDAQ_RPC_INCLUDE_DIR "/usr/include/tirpc")
	find_library(TIRPC_LIBRARY NAMES tirpc)
	if(TIRPC_LIBRARY)
		list(APPEND RCDAQ_RPC_LINK_LIBS ${TIRPC_LIBRARY})
	else()
		list(APPEND RCDAQ_RPC_LINK_LIBS tirpc)
	endif()
endif()

find_path(MOSQUITTO_INCLUDE_DIR mosquitto.h)
if(MOSQUITTO_INCLUDE_DIR)
	find_library(MOSQUITTO_LIBRARY mosquitto)
endif()
set(ENABLE_MQTT OFF)
if(MOSQUITTO_INCLUDE_DIR AND MOSQUITTO_LIBRARY)
	set(ENABLE_MQTT ON)
endif()

find_program(RPCGEN_EXECUTABLE rpcgen REQUIRED)

set(RCDAQ_RPC_X ${CMAKE_CURRENT_SOURCE_DIR}/rcdaq_rpc.x)
set(RCDAQ_RPC_CLNT_C ${CMAKE_CURRENT_BINARY_DIR}/rcdaq_rpc_clnt.c)
set(RCDAQ_RPC_SVC_C ${CMAKE_CURRENT_BINARY_DIR}/rcdaq_rpc_svc.c)
set(RCDAQ_RPC_XDR_C ${CMAKE_CURRENT_BINARY_DIR}/rcdaq_rpc_xdr.c)
set(RCDAQ_RPC_H ${CMAKE_CURRENT_BINARY_DIR}/rcdaq_rpc.h)

add_custom_command(
	OUTPUT ${RCDAQ_RPC_CLNT_C}
	COMMAND ${RPCGEN_EXECUTABLE} -l rcdaq_rpc.x > ${RCDAQ_RPC_CLNT_C}
	DEPENDS ${RCDAQ_RPC_X}
	WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
	VERBATIM
)

add_custom_command(
	OUTPUT ${RCDAQ_RPC_SVC_C}
	COMMAND ${RPCGEN_EXECUTABLE} -m rcdaq_rpc.x > ${RCDAQ_RPC_SVC_C}
	DEPENDS ${RCDAQ_RPC_X}
	WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
	VERBATIM
)

add_custom_command(
	OUTPUT ${RCDAQ_RPC_XDR_C}
	COMMAND ${RPCGEN_EXECUTABLE} -c rcdaq_rpc.x > ${RCDAQ_RPC_XDR_C}
	DEPENDS ${RCDAQ_RPC_X}
	WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
	VERBATIM
)

add_custom_command(
	OUTPUT ${RCDAQ_RPC_H}
	COMMAND ${RPCGEN_EXECUTABLE} -h rcdaq_rpc.x > ${RCDAQ_RPC_H}
	DEPENDS ${RCDAQ_RPC_X}
	WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
	VERBATIM
)

add_custom_target(rcdaq_rpc_generated
	DEPENDS ${RCDAQ_RPC_CLNT_C} ${RCDAQ_RPC_SVC_C} ${RCDAQ_RPC_XDR_C} ${RCDAQ_RPC_H}
)

set(RCDAQ_COMMON_INCLUDE_DIRS
	${CMAKE_CURRENT_SOURCE_DIR}/include
	${CMAKE_CURRENT_SOURCE_DIR}/include/rcdaq
	${CMAKE_CURRENT_SOURCE_DIR}/include/shared
	${CMAKE_CURRENT_SOURCE_DIR}/include/plugins/example
	${CMAKE_CURRENT_SOURCE_DIR}/include/plugins/gauss
	${CMAKE_CURRENT_SOURCE_DIR}/include/apps/common
	${CMAKE_CURRENT_BINARY_DIR}
)
if(RCDAQ_RPC_INCLUDE_DIR)
	list(APPEND RCDAQ_COMMON_INCLUDE_DIRS ${RCDAQ_RPC_INCLUDE_DIR})
endif()

set(RCDAQ_COMMON_LINK_LIBS
	Threads::Threads
)

file(GLOB RCDAQ_BASE_SOURCES CONFIGURE_DEPENDS
	${CMAKE_CURRENT_SOURCE_DIR}/src/rcdaq/*.cc
	${CMAKE_CURRENT_SOURCE_DIR}/src/shared/*.cc
)
if(NOT ENABLE_MQTT)
	list(FILTER RCDAQ_BASE_SOURCES EXCLUDE REGEX ".*/MQTTConnection\\.cc$")
endif()

add_library(rcdaq SHARED ${RCDAQ_BASE_SOURCES})
add_dependencies(rcdaq rcdaq_rpc_generated)
target_include_directories(rcdaq
	PUBLIC
		$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
		$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include/rcdaq>
		$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include/shared>
		$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include/plugins/example>
		$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include/plugins/gauss>
		$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include/apps/common>
		$<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
		$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
	PRIVATE
		${RCDAQ_RPC_INCLUDE_DIR}
)
target_link_libraries(rcdaq PRIVATE ${RCDAQ_COMMON_LINK_LIBS})
target_compile_definitions(rcdaq PRIVATE
	HTMLFILE=\"control.html\"
	HTMLDIR=\"${CMAKE_INSTALL_FULL_DATADIR}/html\"
)

add_library(rcdaqutils SHARED
	src/shared/parseargument.cc
	${RCDAQ_RPC_CLNT_C}
	${RCDAQ_RPC_XDR_C}
)
add_dependencies(rcdaqutils rcdaq_rpc_generated)
target_include_directories(rcdaqutils PRIVATE ${RCDAQ_COMMON_INCLUDE_DIRS})
target_link_libraries(rcdaqutils PUBLIC ${RCDAQ_COMMON_LINK_LIBS} ${RCDAQ_RPC_LINK_LIBS})

file(GLOB RCDAQPLUGIN_EXAMPLE_SOURCES CONFIGURE_DEPENDS
	${CMAKE_CURRENT_SOURCE_DIR}/src/plugins/example/*.cc
)
add_library(rcdaqplugin_example SHARED ${RCDAQPLUGIN_EXAMPLE_SOURCES})
add_dependencies(rcdaqplugin_example rcdaq_rpc_generated)
target_include_directories(rcdaqplugin_example PRIVATE ${RCDAQ_COMMON_INCLUDE_DIRS})

file(GLOB RCDAQPLUGIN_GAUSS_SOURCES CONFIGURE_DEPENDS
	${CMAKE_CURRENT_SOURCE_DIR}/src/plugins/gauss/*.cc
)
add_library(rcdaqplugin_gauss SHARED ${RCDAQPLUGIN_GAUSS_SOURCES})
add_dependencies(rcdaqplugin_gauss rcdaq_rpc_generated)
target_include_directories(rcdaqplugin_gauss PRIVATE ${RCDAQ_COMMON_INCLUDE_DIRS})

add_library(eloghandler_obj OBJECT src/apps/common/eloghandler.cc)
target_include_directories(eloghandler_obj PRIVATE ${RCDAQ_COMMON_INCLUDE_DIRS})

add_executable(elogtest src/apps/elogtest.cc)
target_include_directories(elogtest PRIVATE ${RCDAQ_COMMON_INCLUDE_DIRS})
target_link_libraries(elogtest PRIVATE $<TARGET_OBJECTS:eloghandler_obj>)

add_executable(rcdaq_client src/apps/rcdaq_client.cc)
add_dependencies(rcdaq_client rcdaq_rpc_generated)
target_include_directories(rcdaq_client PRIVATE ${RCDAQ_COMMON_INCLUDE_DIRS})
target_link_libraries(rcdaq_client PRIVATE rcdaqutils ${RCDAQ_COMMON_LINK_LIBS} ${RCDAQ_RPC_LINK_LIBS})

add_executable(rcdaq_server
	src/apps/rcdaq_server.cc
	${RCDAQ_RPC_SVC_C}
	${RCDAQ_RPC_XDR_C}
)
add_dependencies(rcdaq_server rcdaq_rpc_generated)
target_include_directories(rcdaq_server PRIVATE ${RCDAQ_COMMON_INCLUDE_DIRS})
target_link_libraries(rcdaq_server PRIVATE
	rcdaq
	$<TARGET_OBJECTS:eloghandler_obj>
	${RCDAQ_COMMON_LINK_LIBS}
	${RCDAQ_RPC_LINK_LIBS}
	${CMAKE_DL_LIBS}
	lzo2
)
if(ENABLE_MQTT)
	target_include_directories(rcdaq_server PRIVATE ${MOSQUITTO_INCLUDE_DIR})
	target_link_libraries(rcdaq_server PRIVATE ${MOSQUITTO_LIBRARY})
endif()
target_link_options(rcdaq_server PRIVATE -Wl,-export-dynamic)

add_executable(sfs src/apps/sfs.cc)
target_link_libraries(sfs PRIVATE Threads::Threads)

if(WRITEPRDF)
	target_compile_definitions(rcdaq PRIVATE WRITEPRDF=1)
	target_compile_definitions(rcdaqutils PRIVATE WRITEPRDF=1)
	target_compile_definitions(rcdaqplugin_example PRIVATE WRITEPRDF=1)
	target_compile_definitions(rcdaqplugin_gauss PRIVATE WRITEPRDF=1)
	target_compile_definitions(rcdaq_client PRIVATE WRITEPRDF=1)
	target_compile_definitions(rcdaq_server PRIVATE WRITEPRDF=1)
	target_compile_definitions(elogtest PRIVATE WRITEPRDF=1)
	target_compile_definitions(sfs PRIVATE WRITEPRDF=1)
endif()

set(PUBLIC_HEADERS
	include/rcdaq/SubevtStructures.h
	include/rcdaq/SubevtConstants.h
	include/rcdaq/daq_device.h
	include/rcdaq/rcdaq_actions.h
	include/rcdaq/TriggerHandler.h
	include/rcdaq/rcdaq_plugin.h
	include/shared/parseargument.h
)

install(TARGETS
	rcdaq
	rcdaqutils
	rcdaqplugin_example
	rcdaqplugin_gauss
	rcdaq_client
	rcdaq_server
	elogtest
	sfs
	RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
	LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
	ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
)

install(FILES ${PUBLIC_HEADERS} ${RCDAQ_RPC_H}
	DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

file(GLOB SCRIPTS ${CMAKE_CURRENT_SOURCE_DIR}/scripts/*)
install(PROGRAMS ${SCRIPTS} DESTINATION bin)

file(GLOB HTML ${CMAKE_CURRENT_SOURCE_DIR}/html/*)
install(FILES ${HTML} DESTINATION share/html )

