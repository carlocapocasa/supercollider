find_package(Sndfile)
if (WIN32)
	find_package(Portaudio)
endif()


# Here we work out which audio API to use, from system type and/or user option.
if(AUDIOAPI STREQUAL "default")
	if(APPLE)
		set(AUDIOAPI coreaudio)
	elseif(WIN32)
		set(AUDIOAPI portaudio)
	else()
		set(AUDIOAPI jack)
	endif(APPLE)
endif()

if(NOT AUDIOAPI MATCHES "^(jack|coreaudio|portaudio)$")
	message(WARNING "Unrecognised audio API: ${AUDIOAPI}")
endif()

if(AUDIOAPI STREQUAL jack)
	find_package(Jack)
	if(NOT JACK_FOUND)
		message(FATAL_ERROR "Jack selected as audio API, but development files not found")
	endif()
elseif(AUDIOAPI STREQUAL portaudio)
	find_package(Portaudio)
	if(NOT PORTAUDIO_FOUND)
		message(FATAL_ERROR "Portaudio selected as audio API, but development files not found")
	endif()
endif()
message(STATUS "Audio API: ${AUDIOAPI}")

if (NOT Boost_FOUND)
	file(GLOB boost_system_sources ../../external_libraries/boost/libs/system/src/*cpp)
endif()

set(scsynth_sources
	SC_BufGen.cpp

	SC_ComPort.cpp
	SC_CoreAudio.cpp
	SC_Graph.cpp
	SC_GraphDef.cpp
	SC_Group.cpp
	SC_HiddenWorld.h
	SC_Lib_Cintf.cpp
	SC_Lib.cpp
	SC_MiscCmds.cpp
	SC_Node.cpp
	SC_Rate.cpp
	SC_SequencedCommand.cpp
	SC_Str4.cpp
	SC_Unit.cpp
	SC_UnitDef.cpp
	SC_World.cpp
	Rendezvous.cpp

	${CMAKE_SOURCE_DIR}/common/SC_fftlib.cpp
	${CMAKE_SOURCE_DIR}/common/SC_AllocPool.cpp
	${CMAKE_SOURCE_DIR}/common/SC_DirUtils.cpp
	${CMAKE_SOURCE_DIR}/common/SC_Errors.cpp
	${CMAKE_SOURCE_DIR}/common/SC_Reply.cpp
	${CMAKE_SOURCE_DIR}/common/SC_StandAloneInfo_Darwin.cpp
	${CMAKE_SOURCE_DIR}/common/SC_StringBuffer.cpp
	${CMAKE_SOURCE_DIR}/common/SC_StringParser.cpp
	${CMAKE_SOURCE_DIR}/common/Samp.cpp
	${CMAKE_SOURCE_DIR}/common/sc_popen.cpp

	${boost_system_sources}
)

if(APPLE)
	set_property(SOURCE ${CMAKE_SOURCE_DIR}/common/SC_DirUtils.cpp PROPERTY COMPILE_FLAGS -xobjective-c++)
endif()

if(WIN32)
	list(APPEND scsynth_sources ${CMAKE_SOURCE_DIR}/common/SC_Win32Utils.cpp)
endif()

if (FFT_GREEN)
	list(APPEND scsynth_sources ../../common/fftlib.c)
endif()


include_directories(${CMAKE_SOURCE_DIR}/include/common
                    ${CMAKE_SOURCE_DIR}/common
                    ${CMAKE_SOURCE_DIR}/include/server
                    ${CMAKE_SOURCE_DIR}/include/plugin_interface
                    .
)

include_directories (${CMAKE_SOURCE_DIR}/external_libraries/boost_sync/include)

if (AUDIOAPI STREQUAL jack)
	list(APPEND scsynth_sources SC_Jack.cpp)
	add_definitions("-DSC_AUDIO_API=SC_AUDIO_API_JACK" ${JACK_DEFINITIONS})
	include_directories(${JACK_INCLUDE_DIRS})
elseif (AUDIOAPI STREQUAL portaudio)
	list(APPEND scsynth_sources SC_PortAudio.cpp)
	add_definitions("-DSC_AUDIO_API=SC_AUDIO_API_PORTAUDIO" ${PORTAUDIO_DEFINITIONS})
	include_directories(${PORTAUDIO_INCLUDE_DIRS})
endif()

set (FINAL_BUILD 0) # disable final build for scsynth

if (LIBSCSYNTH)
	set (LIBSCSYNTH_TYPE SHARED)
else()
	set (LIBSCSYNTH_TYPE STATIC)
endif()

if (FINAL_BUILD)
	CREATE_FINAL_FILE(libscsynth_final.cpp ${scsynth_sources})
	add_library(libscsynth ${LIBSCSYNTH_TYPE} libscsynth_final.cpp)
else()
	add_library(libscsynth ${LIBSCSYNTH_TYPE} ${scsynth_sources})
endif()

target_compile_definitions(libscsynth PRIVATE BUILDING_SUPERCOLLIDER)
target_compile_definitions(libscsynth PUBLIC  SC_MEMORY_ALIGNMENT=32)

target_link_libraries(libscsynth tlsf)

find_library(DL NAMES dl)
if(DL)
    target_link_libraries(libscsynth ${DL})
endif()

if(NOVA_SIMD)
	target_compile_definitions(libscsynth PUBLIC NOVA_SIMD)
endif()

if(SNDFILE_FOUND)
	target_link_libraries(libscsynth ${SNDFILE_LIBRARIES})
	target_include_directories(libscsynth PUBLIC ${SNDFILE_INCLUDE_DIR})
elseif(NOT NO_LIBSNDFILE)
	message(SEND_ERROR "Cannot find libsndfile")
endif(SNDFILE_FOUND)

if(CMAKE_SYSTEM_NAME MATCHES "Linux" OR CMAKE_SYSTEM_NAME MATCHES "FreeBSD" OR CMAKE_SYSTEM_NAME MATCHES "OpenBSD")
	target_compile_definitions(libscsynth PUBLIC "SC_PLUGIN_DIR=\"${CMAKE_INSTALL_PREFIX}/lib/SuperCollider/plugins\"")
endif()


if(NOT NO_AVAHI)
	if(APPLE)
		target_compile_definitions(libscsynth PUBLIC USE_RENDEZVOUS=1)
	else()
		find_package(Avahi)
		if(AVAHI_FOUND)
			target_compile_definitions(libscsynth PUBLIC USE_RENDEZVOUS=1 HAVE_AVAHI=1)
			target_link_libraries(libscsynth ${AVAHI_LIBRARIES})
			target_include_directories(libscsynth PUBLIC ${AVAHI_INCLUDE_DIRS})
		endif()
	endif()
endif()


if (AUDIOAPI STREQUAL jack)
	target_link_libraries(libscsynth ${JACK_LIBRARIES})
elseif(AUDIOAPI STREQUAL portaudio)
	target_link_libraries(libscsynth ${PORTAUDIO_LIBRARIES})
elseif(AUDIOAPI STREQUAL coreaudio)
	target_link_libraries(libscsynth "-framework CoreAudio")
endif()

if (Boost_FOUND)
	target_link_libraries(libscsynth ${Boost_THREAD_LIBRARY} ${Boost_SYSTEM_LIBRARY})
else()
	target_link_libraries(libscsynth boost_thread boost_system)
endif()

if (WIN32)
	target_link_libraries(libscsynth wsock32 ws2_32 winmm)
endif()


set_property(TARGET libscsynth PROPERTY OUTPUT_NAME scsynth)

if (LIBSCSYNTH)
	# These two properties are ABI version info, not sc version:
	set_property(TARGET libscsynth PROPERTY VERSION     1.0.0)
	set_property(TARGET libscsynth PROPERTY SOVERSION   1)
endif()

if (FFTW3F_FOUND)
	target_include_directories(libscsynth PUBLIC ${FFTW3F_INCLUDE_DIR})
	target_link_libraries(libscsynth ${FFTW3F_LIBRARY})
endif()

if (APPLE)
	target_link_libraries(libscsynth "-framework Accelerate -framework CoreServices -framework Foundation")
endif()

if(CMAKE_SYSTEM_NAME MATCHES "Linux")
	target_link_libraries(libscsynth rt)
endif()

file(GLOB_RECURSE all_headers ../../*hpp)

add_executable(scsynth scsynth_main.cpp ${all_headers})
target_link_libraries(scsynth libscsynth)

if (PTHREADS_FOUND)
    target_link_libraries(scsynth ${PTHREADS_LIBRARIES})
endif()

if(LTO)
    set_property(TARGET scsynth libscsynth
                 APPEND PROPERTY COMPILE_FLAGS "-flto -flto-report")

    set_property(TARGET scsynth
                 APPEND PROPERTY LINK_FLAGS "-flto -flto-report -fwhole-program")

    set_property(TARGET libscsynth
                 APPEND PROPERTY LINK_FLAGS "-flto -flto-report")
endif()

if (LIBSCSYNTH)
	set(INSTALL_TARGETS scsynth libscsynth)
else()
	set(INSTALL_TARGETS scsynth)
endif()

if(APPLE)
	# TODO allow apple users to install to fhs-ish path if INSTALL_FHS option is enabled
	# determines the app name and app install location (scappbundlename, scappdir):
	include (${CMAKE_SOURCE_DIR}/cmake_modules/MacAppFolder.cmake)
	install(TARGETS ${INSTALL_TARGETS}
			RUNTIME DESTINATION "${scappauxresourcesdir}"
			LIBRARY DESTINATION "${scappauxresourcesdir}"
			PERMISSIONS OWNER_READ OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE OWNER_WRITE)
elseif(WIN32)
	install(TARGETS ${INSTALL_TARGETS}
			DESTINATION "SuperCollider"
			PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE)
else()
	install(TARGETS ${INSTALL_TARGETS}
			RUNTIME DESTINATION "bin"
			LIBRARY DESTINATION "lib"
			PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE)
endif()
