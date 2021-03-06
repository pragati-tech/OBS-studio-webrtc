project(obs-outputs)

set(WITH_RTMPS AUTO CACHE STRING "Enable RTMPS support with mbedTLS")
set_property(CACHE WITH_RTMPS PROPERTY STRINGS AUTO ON OFF)

option(STATIC_MBEDTLS "Statically link mbedTLS into binary" OFF)

if (WITH_RTMPS OR (WITH_RTMPS STREQUAL "AUTO"))
	find_package(MbedTLS QUIET)
	find_package(ZLIB QUIET)
endif()

if (LIBMBEDTLS_FOUND AND ZLIB_FOUND)
	add_definitions(-DCRYPTO -DUSE_MBEDTLS)
	include_directories(${LIBMBEDTLS_INCLUDE_DIRS} ${ZLIB_INCLUDE_DIRS})
else()
	if(WITH_RTMPS STREQUAL "AUTO")
		message(WARNING "mbedTLS was not found, RTMPS will be auto-disabled")
	elseif (WITH_RTMPS)
		message(FATAL_ERROR "RTMPS enabled by user, but mbedTLS was not found")
	endif()
	unset(LIBMBEDTLS_LIBRARIES)
	unset(ZLIB_LIBRARIES)
	add_definitions(-DNO_CRYPTO)
endif()

set(COMPILE_FTL FALSE)

configure_file(
	"${CMAKE_CURRENT_SOURCE_DIR}/obs-outputs-config.h.in"
	"${CMAKE_BINARY_DIR}/plugins/obs-outputs/config/obs-outputs-config.h")

include_directories("${CMAKE_BINARY_DIR}/plugins/obs-outputs/config")
include_directories(${CMAKE_SOURCE_DIR}/libobs)

#------------------------------------------------------------------------
# WebRTC
#
if(WIN32)
	set(WEBRTC_USE_FILE_INCLUDED 0)
endif()

# NOTE ALEX: force H264 component here for EBS.
message( STATUS "Looking for libwebrtc" )
find_package(LibWebRTC 79 REQUIRED)
message( STATUS "Looking for libwebrtc - found" )
if( WEBRTC_H264_FOUND )
        message( STATUS "This libwebrtc package was built with H264 support." )
endif()
if( WEBRTC_EXT_OPENSSL_FOUND )
        message( STATUS "This libwebrtc package was built against openssl." )
endif()
if( WEBRTC_EXT_LIBSRTP_FOUND )
        message( STATUS "This libwebrtc package was built with external libsrtp." )
endif()
if( WEBRTC_SFRAME_FOUND )
        message( STATUS "This libwebrtc package was build with SFrame end-to-end encryption." )
endif()

if(WIN32)
	set(obs-outputs_PLATFORM_DEPS
		ws2_32
		winmm
		Iphlpapi)

	if (WITH_RTMPS OR (WITH_RTMPS STREQUAL "AUTO"))
		SET(obs-outputs_PLATFORM_DEPS
			${obs-outputs_PLATFORM_DEPS}
			crypt32)
	endif()
endif()

if(MSVC)
	set(obs-outputs_PLATFORM_DEPS
		${obs-outputs_PLATFORM_DEPS}
		w32-pthreads)
endif()

if(APPLE AND (WITH_RTMPS OR (WITH_RTMPS STREQUAL "AUTO")))
	find_library(FOUNDATION_FRAMEWORK Foundation)
	find_library(SECURITY_FRAMEWORK Security)

	set(obs-outputs_PLATFORM_DEPS
		${obs-outputs_PLATFORM_DEPS}
		${FOUNDATION_FRAMEWORK}
		${SECURITY_FRAMEWORK})
endif()

set(obs-outputs_librtmp_HEADERS
	librtmp/amf.h
	librtmp/bytes.h
	librtmp/cencode.h
	librtmp/dh.h
	librtmp/dhgroups.h
	librtmp/handshake.h
	librtmp/http.h
	librtmp/log.h
	librtmp/md5.h
	librtmp/rtmp.h
	librtmp/rtmp_sys.h)
set(obs-outputs_librtmp_SOURCES
	librtmp/amf.c
	librtmp/cencode.c
	librtmp/hashswf.c
	librtmp/log.c
	# librtmp/md5.c
	librtmp/parseurl.c
	librtmp/rtmp.c)

if(NOT WIN32)
	set_source_files_properties(${obs-outputs_librtmp_SOURCES} PROPERTIES
		COMPILE_FLAGS "-fvisibility=hidden")
endif()

set(obs-outputs_webrtc_HEADERS
	AudioDeviceModuleWrapper.h
	SDPModif.h
	VideoCapturer.h
	WebRTCStream.h
	janus-stream.h
	wowza-stream.h
	millicast-stream.h
	evercast-stream.h)
set(obs-outputs_webrtc_SOURCES
	AudioDeviceModuleWrapper.cpp
	VideoCapturer.cpp
	WebRTCStream.cpp
	janus-stream.cpp
	wowza-stream.cpp
	millicast-stream.cpp
	evercast-stream.cpp)

if(NOT WIN32)
	set_source_files_properties(${obs-outputs_webrtc_SOURCES} PROPERTIES
		COMPILE_FLAGS "-fvisibility=hidden")
endif()

set(obs-outputs_HEADERS
	"${CMAKE_BINARY_DIR}/plugins/obs-outputs/config/obs-outputs-config.h"
	obs-output-ver.h
	rtmp-helpers.h
	rtmp-stream.h
	net-if.h
	flv-mux.h)
set(obs-outputs_SOURCES
	obs-outputs.c
	null-output.c
	rtmp-stream.c
	rtmp-windows.c
	flv-output.c
	flv-mux.c
	net-if.c)

add_library(obs-outputs MODULE
	${ftl_SOURCES}
	${ftl_HEADERS}
	${obs-outputs_SOURCES}
	${obs-outputs_HEADERS}
	${obs-outputs_librtmp_SOURCES}
	${obs-outputs_librtmp_HEADERS}
	${obs-outputs_webrtc_SOURCES}
	${obs-outputs_webrtc_HEADERS})

target_include_directories(obs-outputs
	PUBLIC ${websocketclient_SOURCE_DIR}
	PUBLIC "${CMAKE_CURRENT_LIST_DIR}/../websocket-client") # for WebsocketClient.h

target_link_libraries(obs-outputs
	libobs
	websocketclient
	${LIBMBEDTLS_LIBRARIES}
	${WEBRTC_LIBRARIES}
	${ZLIB_LIBRARIES}
	${ftl_IMPORTS}
	${obs-outputs_PLATFORM_DEPS})

if(WIN32 AND MSVC)
	if(${CMAKE_BUILD_TYPE} STREQUAL "Debug")
		target_compile_options(obs-outputs PUBLIC "/MTd")
	else()
		target_compile_options(obs-outputs PUBLIC "/MT")
	endif()
endif()

install_obs_plugin_with_data(obs-outputs data)
