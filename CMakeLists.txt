#------------------------------------------------------------------------
# ET: Legacy, http://www.etlegacy.com
# - Wolfenstein: Enemy Territory 2.60b compatible client/server
# - based on raedwulf-et: https://bitbucket.org/tcmreastwood/raedwulf-et
#
# Please use TABs to indent! (x4)
#------------------------------------------------------------------------
cmake_minimum_required(VERSION 3.3.2)

project(ETLEGACY C CXX)

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake")
include(GetGitRevisionDescription)
include(CMakeDependentOption)

# We need to add the macros & Functions before any other includes
include(cmake/ETLFunctions.cmake)

#-----------------------------------------------------------------
# Options
#-----------------------------------------------------------------

option(BUILD_SERVER		"Build the dedicated server executable"							ON)
option(BUILD_CLIENT		"Build the client executable"									ON)
option(BUILD_MOD		"Build the mod libraries"										ON)

option(BUILD_MOD_PK3	"Pack the mod libraries into etl_bin.pk3"						ON)
option(BUILD_PAK3_PK3	"Pack updated game scripts into pak3.pk3"						ON)

option(BUNDLED_LIBS		"Use all available bundled libraries instead of system libs"	ON)

LEG_BUNDLE(SDL SDL)
LEG_BUNDLE(ZLIB Zlib)
LEG_BUNDLE(MINIZIP Minizip)
LEG_BUNDLE(JPEG JPEG)
LEG_BUNDLE(CURL Curl "NOT APPLE")
LEG_BUNDLE(OPENSSL OpenSSL "OFF")
LEG_BUNDLE(LUA Lua)
LEG_BUNDLE(OGG_VORBIS OGG)
LEG_BUNDLE(THEORA Theora)
LEG_BUNDLE(OPENAL OpenAL "NOT APPLE")
LEG_BUNDLE(OPENAL_INCLUDE OpenAL-OSX "APPLE")
LEG_BUNDLE(GLEW Glew)
LEG_BUNDLE(FREETYPE Freetype2)
LEG_BUNDLE(JANSSON Jansson)
LEG_BUNDLE(SQLITE3 SQlite3)

# no support for 32 bit binaries on OpenBSD/amd64
cmake_dependent_option(CROSS_COMPILE32 "Compile 32-bit version of ET:L (64bit is incompatible with 2.60b)" ON "NOT CMAKE_SYSTEM MATCHES OpenBSD*" OFF)

# Optional features
option(FEATURE_CURL				"Enable auto-download support using cURL (client)"		ON)
option(FEATURE_OPENSSL			"Build openssl for curl and authentication"				OFF)
option(FEATURE_OGG_VORBIS		"Enable OGG Vorbis support (client)"					ON)
option(FEATURE_THEORA			"Enable Theora video support (client)"					ON)
option(FEATURE_OPENAL			"Enable OpenAL sound backend (client)"					ON)
option(FEATURE_FREETYPE			"Enable Freetype font library support (client)"			ON)
option(FEATURE_LUA				"Enable Lua support (mod)"								ON)
option(FEATURE_MULTIVIEW		"Enable multiview support (mod)"						ON)
option(FEATURE_EDV				"Enable extended demo viewer support (mod)"				ON)
option(FEATURE_ANTICHEAT		"Enable server side anti-wallhack code (server)"		ON)
option(FEATURE_AUTOUPDATE		"Enable updater which downloads latest ET:L files"		ON)
option(FEATURE_RENDERER2		"Build and use the new renderer"						ON)
option(FEATURE_RENDERER_GLES	"Build and use OpenGL ES renderer"						OFF)
option(FEATURE_IPV6				"Enable IPv6 networking code"							OFF)
option(FEATURE_IRC_CLIENT		"Enable IRC client client side"							ON)
option(FEATURE_IRC_SERVER		"Enable IRC client server side"							ON)
option(RENDERER_DYNAMIC			"Build renderer into a dynamic library"					ON)
option(FEATURE_WINDOWS_CONSOLE	"Build win32 executables with Windows console"			ON)
option(FEATURE_GETTEXT			"Enable localization using tinygettext"					ON)
option(FEATURE_JANSSON			"Enable JSON support (client)"							OFF)
option(FEATURE_SERVERMDX		"Enable MDX support on server side (mod)"				ON)
option(FEATURE_DBMS				"Enable dbms support (sqlite3)"							ON)
option(FEATURE_PAKISOLATION		"Enable custom pak isolation"							ON)
option(FORCE_DEBUG				"Force enable legacy debug macros"						OFF)

cmake_dependent_option(FEATURE_LUASQL	"Enable LuaSQL sqlite3 backend (mod)"			ON "FEATURE_LUA" OFF)
cmake_dependent_option(FEATURE_RATING	"Enable skill rating support (mod)"				ON "FEATURE_DBMS" OFF)
cmake_dependent_option(FEATURE_OMNIBOT	"Enable Omni-bot support (mod)"					ON "WIN32 OR APPLE OR CMAKE_SYSTEM_NAME MATCHES Linux" OFF)

option(INSTALL_EXTRA			"Install extra add-ons (omni-bot, geoip, wolfadmin)."	ON)

cmake_dependent_option(INSTALL_OMNIBOT		"Install Omni-bot"							ON "INSTALL_EXTRA" OFF)
cmake_dependent_option(INSTALL_GEOIP		"Install GeoLite geolocation database"		ON "INSTALL_EXTRA" OFF)
cmake_dependent_option(INSTALL_WOLFADMIN	"Install WolfAdmin enhancement suite"		ON "INSTALL_EXTRA" OFF)

option(ARM						"Build ARM version"										OFF)

set(ET_FS_BASEPATH "" CACHE STRING "Copy required genuine ET files from ET_FS_BASEPATH")
cmake_dependent_option(ET_KEY	"Copy existing etkey file from ET_FS_BASEPATH)"			ON "ET_FS_BASEPATH" OFF)

#-----------------------------------------------------------------
# Setup
#-----------------------------------------------------------------

# Group the VisualStudio projects
set(CMAKE_TARGETS_FOLDER CMake)
set(BUNDLED_TARGETS_FOLDER Bundled)
set_property(GLOBAL PROPERTY USE_FOLDERS ON)
set_property(GLOBAL PROPERTY PREDEFINED_TARGETS_FOLDER ${CMAKE_TARGETS_FOLDER})

# Find and setup programs the build needs
include(cmake/ETLPrograms.cmake)

# Setup some basic helper variables
include(cmake/ETLCommon.cmake)

# Platform specific compiler settings
include(cmake/ETLPlatform.cmake)

# Source globs
include(cmake/ETLSources.cmake)

# Git version header generation
include(cmake/ETLVersion.cmake)

# Use bundled libraries
if(BUNDLED_LIBS)
	include(cmake/ETLBundledLibs.cmake)
endif(BUNDLED_LIBS)

# Set up includes and libraries based on feature options
include(cmake/ETLSetupFeatures.cmake)

# Group the sources (only msvc and xcode now)
include(cmake/ETLSourceGroup.cmake)

#-----------------------------------------------------------------
# Build
#-----------------------------------------------------------------

if(BUILD_CLIENT)
	include(cmake/ETLBuildRenderer.cmake)
	include(cmake/ETLBuildClient.cmake)

	#Since CMake 3.6 you are able to set the Visual Studio startup project
	if(NOT ${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION} LESS 3.6)
		set_property(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY VS_STARTUP_PROJECT etl)
	endif()
endif(BUILD_CLIENT)

if(BUILD_SERVER)
	include(cmake/ETLBuildServer.cmake)
endif(BUILD_SERVER)

if(BUILD_MOD)
	include(cmake/ETLBuildMod.cmake)
endif(BUILD_MOD)

if(BUILD_PAK3_PK3)
	include(cmake/ETLBuildPack.cmake)
endif(BUILD_PAK3_PK3)

#-----------------------------------------------------------------
# Post build
#-----------------------------------------------------------------

# Install third-party add-ons
if(BUILD_MOD AND INSTALL_EXTRA)
	if(INSTALL_OMNIBOT)
		include(cmake/ETLInstallOmniBot.cmake)
	endif(INSTALL_OMNIBOT)
	if(INSTALL_GEOIP)
		include(cmake/ETLInstallGeoIP.cmake)
	endif(INSTALL_GEOIP)
	if(INSTALL_WOLFADMIN)
		include(cmake/ETLInstallWolfAdmin.cmake)
	endif(INSTALL_WOLFADMIN)
endif(BUILD_MOD AND INSTALL_EXTRA)

# Generate install and uninstall packs
include(cmake/ETLInstall.cmake)

# Packaging support
include(cmake/ETLCPack.cmake)

set_target_properties(uninstall PROPERTIES FOLDER ${CMAKE_TARGETS_FOLDER})
