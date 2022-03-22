cmake_minimum_required(VERSION 3.12)
list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake/Modules")

project(Discregrid VERSION 0.0.1)

# Visual studio solution directories.
set_property(GLOBAL PROPERTY USE_FOLDERS on)

# Enable simultaneous compilation of source files.
if(MSVC)
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /MP")
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /openmp")
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /wd4250")
endif(MSVC)

OPTION(BUILD_AS_SHARED_LIBS "Build all the libraries as shared" OFF)
if (BUILD_AS_SHARED_LIBS)
	set(BUILD_AS_SHARED_LIBS ON)
	if(WIN32)
		set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON CACHE BOOL "Export all symbols")
	endif(WIN32)	
endif (BUILD_AS_SHARED_LIBS)


set(HEADERS
	include/Discregrid/discrete_grid.hpp
	include/Discregrid/cubic_lagrange_discrete_grid.hpp
)

set(HEADERS_ACCELERATION

	include/Discregrid/acceleration/bounding_sphere_hierarchy.hpp
	include/Discregrid/acceleration/bounding_sphere.hpp
	include/Discregrid/acceleration/kd_tree.hpp
	include/Discregrid/acceleration/kd_tree.inl
)

set(HEADERS_MESH
	include/Discregrid/mesh/triangle_mesh.hpp
	include/Discregrid/mesh/entity_containers.hpp
	include/Discregrid/mesh/entity_iterators.hpp
	include/Discregrid/mesh/halfedge.hpp
)

set(HEADERS_GEOMETRY
	include/Discregrid/geometry/mesh_distance.hpp

	src/geometry/point_triangle_distance.hpp
)

set(HEADERS_UTILITY
	include/Discregrid/utility/serialize.hpp
	include/Discregrid/utility/lru_cache.hpp

	src/utility/timing.hpp
	src/utility/spinlock.hpp
)

set(SOURCES
	src/discrete_grid.cpp
	src/cubic_lagrange_discrete_grid.cpp
)

set(SOURCES_DATA
)

set(SOURCES_ACCELERATION
	src/acceleration/bounding_sphere_hierarchy.cpp
)

set(SOURCES_MESH
	src/mesh/entity_containers.cpp
	src/mesh/entity_iterators.cpp
	src/mesh/triangle_mesh.cpp
)

set(SOURCES_GEOMETRY
	src/geometry/mesh_distance.cpp
	src/geometry/point_triangle_distance.cpp
)

set(SOURCES_UTILITY
	src/utility/timing.cpp
)

macro(SOURCEGROUP name)
	string(TOLOWER ${name} name_lower)
	string(SUBSTRING ${name_lower} 0 1 FIRST_LETTER)
	string(TOUPPER ${FIRST_LETTER} FIRST_LETTER)
	string(REGEX REPLACE "^.(.*)" "${FIRST_LETTER}\\1" dname "${name_lower}")
	source_group("Source Files ${dname}" FILES ${SOURCES_${name}})
	source_group("Header Files ${dname}" FILES ${HEADERS_${name}})
endmacro()
SOURCEGROUP(MESH)
SOURCEGROUP(ACCELERATION)
SOURCEGROUP(DATA)
SOURCEGROUP(GEOMETRY)
SOURCEGROUP(UTILITY)

# OpenMP support.
if(APPLE)
	include(PatchOpenMPApple)
else()
	find_package(OpenMP REQUIRED)
endif()

if(OPENMP_FOUND)
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
	set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${OpenMP_EXE_LINKER_FLAGS}")
endif()

# Eigen library.
find_package(Eigen3 CONFIG REQUIRED)

set(DISCREGRID_SOURCE_FILES
	${HEADERS}
	${SOURCES}
	${HEADERS_ACCELERATION}
	${SOURCES_ACCELERATION}
	${SOURCES_DATA}
	${HEADERS_MESH}
	${SOURCES_MESH}
	${HEADERS_GEOMETRY}
	${SOURCES_GEOMETRY}
	${HEADERS_UTILITY}
	${SOURCES_UTILITY}
)

if(BUILD_AS_SHARED_LIBS)
	add_library(Discregrid SHARED)
else()
	add_library(Discregrid)
endif()

# Disable stupid MSVC compiler warnings.
if(WIN32)
	target_compile_definitions(Discregrid
	PRIVATE
		-D_SCL_SECURE_NO_WARNINGS
		-D_CRT_SECURE_NO_WARNINGS
		-DNOMINMAX
		-D_USE_MATH_DEFINES
	)
endif(WIN32)

target_sources(Discregrid 
PRIVATE
	${DISCREGRID_SOURCE_FILES}
)

target_include_directories(Discregrid
PUBLIC
	$<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/include>
INTERFACE
	$<INSTALL_INTERFACE:include>
PRIVATE
	"${CMAKE_CURRENT_LIST_DIR}/include/Discregrid"
	"${CMAKE_CURRENT_LIST_DIR}/extern"
)

target_link_libraries(Discregrid
PUBLIC 
	Eigen3::Eigen
)

include(GNUInstallDirs)

install(TARGETS Discregrid
		EXPORT DiscregridTargets
		LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
		ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
		RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
		PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
		INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

install(
	DIRECTORY include/
	DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

install(
	EXPORT DiscregridTargets
	FILE DiscregridTargets.cmake
	NAMESPACE Discregrid::
	DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/tmc
)

include(CMakePackageConfigHelpers)
write_basic_package_version_file(
    ${CMAKE_CURRENT_BINARY_DIR}/DiscregridConfigVersion.cmake
    VERSION ${PROJECT_VERSION}
    COMPATIBILITY SameMajorVersion
)

configure_package_config_file(${CMAKE_CURRENT_SOURCE_DIR}/cmake/DiscregridConfig.cmake.in
    ${CMAKE_CURRENT_BINARY_DIR}/DiscregridConfig.cmake
    INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/Discregrid
)

install(
    FILES
        ${CMAKE_CURRENT_BINARY_DIR}/DiscregridConfig.cmake
        ${CMAKE_CURRENT_BINARY_DIR}/DiscregridConfigVersion.cmake
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/Discregrid
)

option(BUILD_CMD_EXECUTABLE "Build command line executable" ON)
if(BUILD_CMD_EXECUTABLE)
	add_subdirectory(cmd)
endif(BUILD_CMD_EXECUTABLE)
