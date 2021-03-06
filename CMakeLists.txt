cmake_minimum_required(VERSION 3.0 FATAL_ERROR)

# Enforce "Debug" as standard build type
if(NOT CMAKE_BUILD_TYPE)
	set(CMAKE_BUILD_TYPE "Debug" CACHE STRING "Choose the type of build, options are: Debug Release RelWithDebInfo MinSizeRel." FORCE)
endif()

# CMake project configuration
project(yquake2)

# Cmake module search path
set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_LIST_DIR}/stuff/cmake/modules ${CMAKE_MODULE_PATH})

if(YQUAKE2LIBS)
	if(CMAKE_CROSSCOMPILING)
		set(CMAKE_FIND_ROOT_PATH ${YQUAKE2LIBS})
	else()
		set(ENV{CMAKE_PREFIX_PATH} ${YQUAKE2LIBS})
	endif()

	set(ENV{OPENALDIR} ${YQUAKE2LIBS})
	set(ENV{SDLDIR} ${YQUAKE2LIBS})
	set(ENV{SDL2DIR} ${YQUAKE2LIBS})
endif()

# Add extended path for FreeBSD and Homebrew on OS X
list(APPEND CMAKE_PREFIX_PATH /usr/local)

# Enforce compiler flags (GCC / Clang compatible, yquake2
# won't build with another compiler anyways)
#  -Wall                -> More warnings
#  -fno-strict-aliasing -> Quake 2 is far away from strict aliasing
#  -fwrapv              -> Make signed integer overflows defined
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -std=gnu99 -Wall -fno-strict-aliasing -fwrapv")

# Use -O2 as maximum optimization level. -O3 has it's problems with yquake2.
string(REPLACE "-O3" "-O2" CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE}")

# yquake2 compilation options
option(ZIP_SUPPORT "ZIP support" ON)
option(OGG_SUPPORT "OGG Vorbis playback support (Music)" ON)
option(OPENAL_SUPPORT "OpenAL support" ON)
option(SYSTEMWIDE_SUPPORT "Enable systemwide installation of game assets" OFF)

# These variables will act as our list of include folders and linker flags
set(yquake2IncludeDirectories)
set(yquake2LinkerDirectories)
set(yquake2LinkerFlags)
set(yquake2ClientLinkerFlags)
set(yquake2RendererLinkerFlags)
set(yquake2SDLLinkerFlags)
set(yquake2ZLibLinkerFlags)

# Set directory locations (allowing us to move directories easily)
set(SOURCE_DIR ${CMAKE_CURRENT_LIST_DIR}/src)
set(BACKENDS_SRC_DIR ${SOURCE_DIR}/backends)
set(COMMON_SRC_DIR ${SOURCE_DIR}/common)
set(GAME_SRC_DIR ${SOURCE_DIR}/game)
set(SERVER_SRC_DIR ${SOURCE_DIR}/server)
set(CLIENT_SRC_DIR ${SOURCE_DIR}/client)
set(GL_SRC_DIR ${SOURCE_DIR}/client/refresh)

# Operating system
set(YQ2OSTYPE "${CMAKE_SYSTEM_NAME}" CACHE STRING "Override operation system type")
add_definitions(-DYQ2OSTYPE="${YQ2OSTYPE}")

# Architecture string
set(YQ2ARCH "${CMAKE_SYSTEM_PROCESSOR}" CACHE STRING "Override CPU architecture")
string(REGEX REPLACE "amd64" "x86_64" ARCH ${YQ2ARCH})
string(REGEX REPLACE "i.86" "i386" ARCH ${ARCH})
string(REGEX REPLACE "^arm.*" "arm" ARCH ${ARCH})
add_definitions(-DYQ2ARCH="${ARCH}")

# Systemwide installation of game assets
if(${SYSTEMWIDE_SUPPORT})
    add_definitions(-DSYSTEMWIDE)
endif()

# Required libraries to build the different components of the binaries. Find
# them and add the include/linker directories and flags (in case the package
# manager find it in a weird place)
find_package(SDL2)
if(${SDL2_FOUND})
	add_definitions(-DSDL2)
	list(APPEND yquake2IncludeDirectories "${SDL2_INCLUDE_DIR}/..")
	list(APPEND yquake2SDLLinkerFlags ${SDL2_LIBRARY})
else()
	find_package(SDL REQUIRED)
	add_definitions(-DWITH_CDA)
	list(APPEND yquake2IncludeDirectories "${SDL_INCLUDE_DIR}/..")
	list(APPEND yquake2SDLLinkerFlags ${SDL_LIBRARY})
endif()

find_package(OpenGL REQUIRED)
list(APPEND yquake2IncludeDirectories ${OPENGL_INCLUDE_DIR})
list(APPEND yquake2RendererLinkerFlags ${OPENGL_LIBRARIES})

if(${ZIP_SUPPORT})
	find_package(ZLIB REQUIRED)
	list(APPEND yquake2IncludeDirectories ${ZLIB_INCLUDE_DIRS})
	list(APPEND yquake2ZLibLinkerFlags ${ZLIB_LIBRARIES})
	add_definitions(-DZIP -DNOUNCRYPT)
endif()

if(${OGG_SUPPORT})
	find_package(OggVorbis)
	if(${OGGVORBIS_FOUND})
		add_definitions(-DOGG)
		list(APPEND yquake2IncludeDirectories ${OGGVORBIS_INCLUDE_DIR})
		list(APPEND yquake2ClientLinkerFlags ${OGG_LIBRARY} ${VORBIS_LIBRARY} ${VORBISFILE_LIBRARY})
	endif()
endif()

if(${OPENAL_SUPPORT})
	find_package(OpenAL)

	# TODO: OS X is still missing here
	if(${OPENAL_FOUND} AND NOT(${CMAKE_SYSTEM_NAME} MATCHES "Darwin"))
		list(APPEND yquake2IncludeDirectories "${OPENAL_INCLUDE_DIR}")
		list(APPEND yquake2ClientLinkerFlags ${OPENAL_LIBRARY})

		if(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
			add_definitions(-DUSE_OPENAL -DDEFAULT_OPENAL_DRIVER="openal32.dll")
		elseif(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
			add_definitions(-DUSE_OPENAL -DDEFAULT_OPENAL_DRIVER="libopenal.dylib")
		elseif((${CMAKE_SYSTEM_NAME} MATCHES "FreeBSD") OR (${CMAKE_SYSTEM_NAME} MATCHES "OpenBSD"))
			add_definitions(-DUSE_OPENAL -DDEFAULT_OPENAL_DRIVER="libopenal.so")
		else()
			add_definitions(-DUSE_OPENAL -DDEFAULT_OPENAL_DRIVER="libopenal.so.1")
		endif()
	endif()
endif()

if(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
	list(APPEND yquake2LinkerFlags "-lm -static-libgcc")
else()
	list(APPEND yquake2LinkerFlags "-lm -rdynamic")
endif()

list(APPEND yquake2LinkerFlags ${CMAKE_DL_LIBS})

# With all of those libraries and user defined paths
# added, lets give them to the compiler and linker.
include_directories(${yquake2IncludeDirectories} ${CMAKE_SOURCE_DIR}/src/client/refresh/gl3/glad/include)
link_directories(${yquake2LinkerDirectories})

set(Backends-Generic-Source
	${BACKENDS_SRC_DIR}/generic/misc.c
	${BACKENDS_SRC_DIR}/generic/qal.c
	${BACKENDS_SRC_DIR}/generic/vid.c

	${BACKENDS_SRC_DIR}/sdl/cd.c
	${BACKENDS_SRC_DIR}/sdl/input.c
	${BACKENDS_SRC_DIR}/sdl/refresh.c
	${BACKENDS_SRC_DIR}/sdl/sound.c
	)

set(Backends-Generic-Header
	${BACKENDS_SRC_DIR}/generic/header/input.h
	${BACKENDS_SRC_DIR}/generic/header/qal.h
	)

set(Backends-Unix-Source
	${BACKENDS_SRC_DIR}/generic/misc.c
	${BACKENDS_SRC_DIR}/unix/main.c
	${BACKENDS_SRC_DIR}/unix/network.c
	${BACKENDS_SRC_DIR}/unix/signalhandler.c
	${BACKENDS_SRC_DIR}/unix/system.c
	${BACKENDS_SRC_DIR}/unix/shared/hunk.c
	)

set(Backends-Unix-Header
	${BACKENDS_SRC_DIR}/unix/header/unix.h
	)

set(Backends-Windows-Source
	${BACKENDS_SRC_DIR}/generic/misc.c
	${BACKENDS_SRC_DIR}/windows/icon.rc
	${BACKENDS_SRC_DIR}/windows/network.c
	${BACKENDS_SRC_DIR}/windows/system.c
	${BACKENDS_SRC_DIR}/windows/shared/mem.c
	)

set(Backends-Windows-Header
	${BACKENDS_SRC_DIR}/windows/header/resource.h
	${BACKENDS_SRC_DIR}/windows/header/winquake.h
	)

set(GL-Windows-Source
	${BACKENDS_SRC_DIR}/windows/shared/mem.c
	)

set(GL-Unix-Source
	${BACKENDS_SRC_DIR}/unix/shared/hunk.c
	)

# Set the nessesary platform specific source
if(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
	set(Platform-Specific-Source ${Backends-Windows-Source} ${Backends-Windows-Header})
	set(GL-Platform-Specific-Source ${GL-Windows-Source})
else()
	set(Platform-Specific-Source ${Backends-Unix-Source} ${Backends-Unix-Header})
	set(GL-Platform-Specific-Source ${GL-Unix-Source})
endif()

set(Game-Source
	${COMMON_SRC_DIR}/shared/flash.c
	${COMMON_SRC_DIR}/shared/rand.c
	${COMMON_SRC_DIR}/shared/shared.c
	${GAME_SRC_DIR}/g_ai.c
	${GAME_SRC_DIR}/g_chase.c
	${GAME_SRC_DIR}/g_cmds.c
	${GAME_SRC_DIR}/g_combat.c
	${GAME_SRC_DIR}/g_func.c
	${GAME_SRC_DIR}/g_items.c
	${GAME_SRC_DIR}/g_main.c
	${GAME_SRC_DIR}/g_misc.c
	${GAME_SRC_DIR}/g_monster.c
	${GAME_SRC_DIR}/g_phys.c
	${GAME_SRC_DIR}/g_spawn.c
	${GAME_SRC_DIR}/g_svcmds.c
	${GAME_SRC_DIR}/g_target.c
	${GAME_SRC_DIR}/g_trigger.c
	${GAME_SRC_DIR}/g_turret.c
	${GAME_SRC_DIR}/g_utils.c
	${GAME_SRC_DIR}/g_weapon.c
	${GAME_SRC_DIR}/monster/berserker/berserker.c
	${GAME_SRC_DIR}/monster/boss2/boss2.c
	${GAME_SRC_DIR}/monster/boss3/boss3.c
	${GAME_SRC_DIR}/monster/boss3/boss31.c
	${GAME_SRC_DIR}/monster/boss3/boss32.c
	${GAME_SRC_DIR}/monster/brain/brain.c
	${GAME_SRC_DIR}/monster/chick/chick.c
	${GAME_SRC_DIR}/monster/flipper/flipper.c
	${GAME_SRC_DIR}/monster/float/float.c
	${GAME_SRC_DIR}/monster/flyer/flyer.c
	${GAME_SRC_DIR}/monster/gladiator/gladiator.c
	${GAME_SRC_DIR}/monster/gunner/gunner.c
	${GAME_SRC_DIR}/monster/hover/hover.c
	${GAME_SRC_DIR}/monster/infantry/infantry.c
	${GAME_SRC_DIR}/monster/insane/insane.c
	${GAME_SRC_DIR}/monster/medic/medic.c
	${GAME_SRC_DIR}/monster/misc/move.c
	${GAME_SRC_DIR}/monster/mutant/mutant.c
	${GAME_SRC_DIR}/monster/parasite/parasite.c
	${GAME_SRC_DIR}/monster/soldier/soldier.c
	${GAME_SRC_DIR}/monster/supertank/supertank.c
	${GAME_SRC_DIR}/monster/tank/tank.c
	${GAME_SRC_DIR}/player/client.c
	${GAME_SRC_DIR}/player/hud.c
	${GAME_SRC_DIR}/player/trail.c
	${GAME_SRC_DIR}/player/view.c
	${GAME_SRC_DIR}/player/weapon.c
	${GAME_SRC_DIR}/savegame/savegame.c
	)

set(Game-Header
	${GAME_SRC_DIR}/header/game.h
	${GAME_SRC_DIR}/header/local.h
	${GAME_SRC_DIR}/monster/berserker/berserker.h
	${GAME_SRC_DIR}/monster/boss2/boss2.h
	${GAME_SRC_DIR}/monster/boss3/boss31.h
	${GAME_SRC_DIR}/monster/boss3/boss32.h
	${GAME_SRC_DIR}/monster/brain/brain.h
	${GAME_SRC_DIR}/monster/chick/chick.h
	${GAME_SRC_DIR}/monster/flipper/flipper.h
	${GAME_SRC_DIR}/monster/float/float.h
	${GAME_SRC_DIR}/monster/flyer/flyer.h
	${GAME_SRC_DIR}/monster/gladiator/gladiator.h
	${GAME_SRC_DIR}/monster/gunner/gunner.h
	${GAME_SRC_DIR}/monster/hover/hover.h
	${GAME_SRC_DIR}/monster/infantry/infantry.h
	${GAME_SRC_DIR}/monster/insane/insane.h
	${GAME_SRC_DIR}/monster/medic/medic.h
	${GAME_SRC_DIR}/monster/misc/player.h
	${GAME_SRC_DIR}/monster/mutant/mutant.h
	${GAME_SRC_DIR}/monster/parasite/parasite.h
	${GAME_SRC_DIR}/monster/soldier/soldier.h
	${GAME_SRC_DIR}/monster/supertank/supertank.h
	${GAME_SRC_DIR}/monster/tank/tank.h
	${GAME_SRC_DIR}/savegame/tables/clientfields.h
	${GAME_SRC_DIR}/savegame/tables/fields.h
	${GAME_SRC_DIR}/savegame/tables/gamefunc_decs.h
	${GAME_SRC_DIR}/savegame/tables/gamefunc_list.h
	${GAME_SRC_DIR}/savegame/tables/gamemmove_decs.h
	${GAME_SRC_DIR}/savegame/tables/gamemmove_list.h
	${GAME_SRC_DIR}/savegame/tables/levelfields.h
	)

set(Client-Source
	${CLIENT_SRC_DIR}/cl_cin.c
	${CLIENT_SRC_DIR}/cl_console.c
	${CLIENT_SRC_DIR}/cl_download.c
	${CLIENT_SRC_DIR}/cl_effects.c
	${CLIENT_SRC_DIR}/cl_entities.c
	${CLIENT_SRC_DIR}/cl_input.c
	${CLIENT_SRC_DIR}/cl_inventory.c
	${CLIENT_SRC_DIR}/cl_keyboard.c
	${CLIENT_SRC_DIR}/cl_lights.c
	${CLIENT_SRC_DIR}/cl_main.c
	${CLIENT_SRC_DIR}/cl_network.c
	${CLIENT_SRC_DIR}/cl_parse.c
	${CLIENT_SRC_DIR}/cl_particles.c
	${CLIENT_SRC_DIR}/cl_prediction.c
	${CLIENT_SRC_DIR}/cl_screen.c
	${CLIENT_SRC_DIR}/cl_tempentities.c
	${CLIENT_SRC_DIR}/cl_view.c
	${CLIENT_SRC_DIR}/menu/menu.c
	${CLIENT_SRC_DIR}/menu/qmenu.c
	${CLIENT_SRC_DIR}/menu/videomenu.c
	${CLIENT_SRC_DIR}/sound/ogg.c
	${CLIENT_SRC_DIR}/sound/openal.c
	${CLIENT_SRC_DIR}/sound/sound.c
	${CLIENT_SRC_DIR}/sound/wave.c
	${COMMON_SRC_DIR}/argproc.c
	${COMMON_SRC_DIR}/clientserver.c
	${COMMON_SRC_DIR}/collision.c
	${COMMON_SRC_DIR}/crc.c
	${COMMON_SRC_DIR}/cmdparser.c
	${COMMON_SRC_DIR}/cvar.c
	${COMMON_SRC_DIR}/filesystem.c
	${COMMON_SRC_DIR}/glob.c
	${COMMON_SRC_DIR}/md4.c
	${COMMON_SRC_DIR}/movemsg.c
	${COMMON_SRC_DIR}/misc.c
	${COMMON_SRC_DIR}/netchan.c
	${COMMON_SRC_DIR}/pmove.c
	${COMMON_SRC_DIR}/szone.c
	${COMMON_SRC_DIR}/zone.c
	${COMMON_SRC_DIR}/shared/flash.c
	${COMMON_SRC_DIR}/shared/rand.c
	${COMMON_SRC_DIR}/shared/shared.c
	${COMMON_SRC_DIR}/unzip/ioapi.c
	${COMMON_SRC_DIR}/unzip/unzip.c
	${SERVER_SRC_DIR}/sv_cmd.c
	${SERVER_SRC_DIR}/sv_conless.c
	${SERVER_SRC_DIR}/sv_entities.c
	${SERVER_SRC_DIR}/sv_game.c
	${SERVER_SRC_DIR}/sv_init.c
	${SERVER_SRC_DIR}/sv_main.c
	${SERVER_SRC_DIR}/sv_save.c
	${SERVER_SRC_DIR}/sv_send.c
	${SERVER_SRC_DIR}/sv_user.c
	${SERVER_SRC_DIR}/sv_world.c
	)

set(Client-Header
	${CLIENT_SRC_DIR}/header/client.h
	${CLIENT_SRC_DIR}/header/console.h
	${CLIENT_SRC_DIR}/header/keyboard.h
	${CLIENT_SRC_DIR}/header/ref.h
	${CLIENT_SRC_DIR}/header/screen.h
	${CLIENT_SRC_DIR}/header/vid.h
	${CLIENT_SRC_DIR}/menu/header/qmenu.h
	${CLIENT_SRC_DIR}/sound/header/cdaudio.h
	${CLIENT_SRC_DIR}/sound/header/local.h
	${CLIENT_SRC_DIR}/sound/header/sound.h
	${CLIENT_SRC_DIR}/sound/header/vorbis.h
	${COMMON_SRC_DIR}/header/common.h
	${COMMON_SRC_DIR}/header/crc.h
	${COMMON_SRC_DIR}/header/files.h
	${COMMON_SRC_DIR}/header/glob.h
	${COMMON_SRC_DIR}/header/shared.h
	${COMMON_SRC_DIR}/header/zone.h
	${COMMON_SRC_DIR}/unzip/ioapi.h
	${COMMON_SRC_DIR}/unzip/unzip.h
	${SERVER_SRC_DIR}/header/server.h
	)

set(Server-Source
	${COMMON_SRC_DIR}/argproc.c
	${COMMON_SRC_DIR}/clientserver.c
	${COMMON_SRC_DIR}/collision.c
	${COMMON_SRC_DIR}/crc.c
	${COMMON_SRC_DIR}/cmdparser.c
	${COMMON_SRC_DIR}/cvar.c
	${COMMON_SRC_DIR}/filesystem.c
	${COMMON_SRC_DIR}/glob.c
	${COMMON_SRC_DIR}/md4.c
	${COMMON_SRC_DIR}/misc.c
	${COMMON_SRC_DIR}/movemsg.c
	${COMMON_SRC_DIR}/netchan.c
	${COMMON_SRC_DIR}/pmove.c
	${COMMON_SRC_DIR}/szone.c
	${COMMON_SRC_DIR}/zone.c
	${COMMON_SRC_DIR}/shared/rand.c
	${COMMON_SRC_DIR}/shared/shared.c
	${COMMON_SRC_DIR}/unzip/ioapi.c
	${COMMON_SRC_DIR}/unzip/unzip.c
	${SERVER_SRC_DIR}/sv_cmd.c
	${SERVER_SRC_DIR}/sv_conless.c
	${SERVER_SRC_DIR}/sv_entities.c
	${SERVER_SRC_DIR}/sv_game.c
	${SERVER_SRC_DIR}/sv_init.c
	${SERVER_SRC_DIR}/sv_main.c
	${SERVER_SRC_DIR}/sv_save.c
	${SERVER_SRC_DIR}/sv_send.c
	${SERVER_SRC_DIR}/sv_user.c
	${SERVER_SRC_DIR}/sv_world.c
	)

set(Server-Header
	${COMMON_SRC_DIR}/header/common.h
	${COMMON_SRC_DIR}/header/crc.h
	${COMMON_SRC_DIR}/header/files.h
	${COMMON_SRC_DIR}/header/glob.h
	${COMMON_SRC_DIR}/header/shared.h
	${COMMON_SRC_DIR}/header/zone.h
	${COMMON_SRC_DIR}/unzip/ioapi.h
	${COMMON_SRC_DIR}/unzip/unzip.h
	${SERVER_SRC_DIR}/header/server.h
	)

set(GL1-Source
	${GL_SRC_DIR}/gl/qgl.c
	${GL_SRC_DIR}/gl/r_draw.c
	${GL_SRC_DIR}/gl/r_image.c
	${GL_SRC_DIR}/gl/r_light.c
	${GL_SRC_DIR}/gl/r_lightmap.c
	${GL_SRC_DIR}/gl/r_main.c
	${GL_SRC_DIR}/gl/r_mesh.c
	${GL_SRC_DIR}/gl/r_misc.c
	${GL_SRC_DIR}/gl/r_model.c
	${GL_SRC_DIR}/gl/r_scrap.c
	${GL_SRC_DIR}/gl/r_surf.c
	${GL_SRC_DIR}/gl/r_warp.c
	${GL_SRC_DIR}/gl/r_sdl.c
	${GL_SRC_DIR}/gl/r_md2.c
	${GL_SRC_DIR}/gl/r_sp2.c
	${GL_SRC_DIR}/files/pcx.c
	${GL_SRC_DIR}/files/stb.c
	${GL_SRC_DIR}/files/wal.c
	${COMMON_SRC_DIR}/shared/shared.c
	${COMMON_SRC_DIR}/md4.c
	)

set(GL1-Header
	${GL_SRC_DIR}/ref_shared.h
	${GL_SRC_DIR}/constants/anorms.h
	${GL_SRC_DIR}/constants/anormtab.h
	${GL_SRC_DIR}/constants/warpsin.h
	${GL_SRC_DIR}/files/stb_image.h
	${GL_SRC_DIR}/gl/header/local.h
	${GL_SRC_DIR}/gl/header/model.h
	${GL_SRC_DIR}/gl/header/qgl.h
	${COMMON_SRC_DIR}/header/shared.h
	)

set(GL3-Source
	${GL_SRC_DIR}/gl3/gl3_draw.c
	${GL_SRC_DIR}/gl3/gl3_image.c
	${GL_SRC_DIR}/gl3/gl3_light.c
	${GL_SRC_DIR}/gl3/gl3_lightmap.c
	${GL_SRC_DIR}/gl3/gl3_main.c
	${GL_SRC_DIR}/gl3/gl3_mesh.c
	${GL_SRC_DIR}/gl3/gl3_misc.c
	${GL_SRC_DIR}/gl3/gl3_model.c
	${GL_SRC_DIR}/gl3/gl3_sdl.c
	${GL_SRC_DIR}/gl3/gl3_surf.c
	${GL_SRC_DIR}/gl3/gl3_warp.c
	${GL_SRC_DIR}/gl3/gl3_shaders.c
	${GL_SRC_DIR}/gl3/gl3_md2.c
	${GL_SRC_DIR}/gl3/gl3_sp2.c
	${GL_SRC_DIR}/gl3/glad/src/glad.c
	${GL_SRC_DIR}/files/pcx.c
	${GL_SRC_DIR}/files/stb.c
	${GL_SRC_DIR}/files/wal.c
	${COMMON_SRC_DIR}/shared/shared.c
	${COMMON_SRC_DIR}/md4.c
	)

set(GL3-Header
	${GL_SRC_DIR}/ref_shared.h
	${GL_SRC_DIR}/constants/anorms.h
	${GL_SRC_DIR}/constants/anormtab.h
	${GL_SRC_DIR}/constants/warpsin.h
	${GL_SRC_DIR}/files/stb_image.h
	${GL_SRC_DIR}/gl3/glad/include/glad/glad.h
	${GL_SRC_DIR}/gl3/glad/include/KHR/khrplatform.h
	${GL_SRC_DIR}/gl3/header/DG_dynarr.h
	${GL_SRC_DIR}/gl3/header/HandmadeMath.h
	${GL_SRC_DIR}/gl3/header/local.h
	${GL_SRC_DIR}/gl3/header/model.h
	${COMMON_SRC_DIR}/header/shared.h
	)

# Main Quake 2 executable
add_executable(quake2 ${Client-Source} ${Client-Header} ${Platform-Specific-Source}
		${Backends-Generic-Source} ${Backends-Generic-Header})
set_target_properties(quake2 PROPERTIES
	RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/release
	)
if(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
	target_link_libraries(quake2 ${yquake2LinkerFlags} ${yquake2ClientLinkerFlags}
			${yquake2SDLLinkerFlags} ${yquake2ZLibLinkerFlags} ws2_32 winmm)
else()
	target_link_libraries(quake2 ${yquake2LinkerFlags} ${yquake2ClientLinkerFlags}
			${yquake2SDLLinkerFlags} ${yquake2ZLibLinkerFlags})
endif()

# Quake 2 Dedicated Server
add_executable(q2ded ${Server-Source} ${Server-Header} ${Platform-Specific-Source})
set_target_properties(q2ded PROPERTIES
	COMPILE_DEFINITIONS "DEDICATED_ONLY"
	RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/release
	)
if(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
	target_link_libraries(q2ded ${yquake2LinkerFlags} ${yquake2ZLibLinkerFlags} ws2_32 winmm)
else()
	target_link_libraries(q2ded ${yquake2LinkerFlags} ${yquake2ZLibLinkerFlags})
endif()

# Build the game dynamic library
add_library(game MODULE ${Game-Source} ${Game-Header})
set_target_properties(game PROPERTIES
		PREFIX ""
		LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/release/baseq2
		RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/release/baseq2
		)
target_link_libraries(game ${yquake2LinkerFlags})

# Build the GL1 dynamic library
add_library(ref_gl1 MODULE ${GL1-Source} ${GL1-Header} ${GL-Platform-Specific-Source})
set_target_properties(ref_gl1 PROPERTIES
		PREFIX ""
		LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/release
		RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/release
		)
target_link_libraries(ref_gl1 ${yquake2LinkerFlags} ${yquake2OpenGLLinkerFlags} ${yquake2SDLLinkerFlags})

# Build the GL3 dynamic library
add_library(ref_gl3 MODULE ${GL3-Source} ${GL3-Header} ${GL-Platform-Specific-Source})
set_target_properties(ref_gl3 PROPERTIES
		PREFIX ""
		LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/release
		RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/release
		)
target_link_libraries(ref_gl3 ${yquake2LinkerFlags} ${yquake2OpenGLLinkerFlags} ${yquake2SDLLinkerFlags})
