# Eigen library.
find_package(Eigen3 CONFIG REQUIRED)

add_executable(GenerateDensityMap)

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

if(WIN32)
	target_compile_definitions(GenerateDensityMap
	PRIVATE
		-D_SCL_SECURE_NO_WARNINGS
		-D_USE_MATH_DEFINES
	)
endif(WIN32)

add_dependencies(GenerateDensityMap
	Discregrid
)

target_sources(GenerateDensityMap
PRIVATE
	main.cpp
	gauss_quadrature.hpp
	gauss_quadrature.cpp
	sph_kernel.hpp
)

target_include_directories(GenerateDensityMap
PRIVATE
	../../extern
	../../discregrid/include
)

target_link_libraries(GenerateDensityMap
PRIVATE
	Discregrid
	Eigen3::Eigen
)

set_target_properties(GenerateDensityMap PROPERTIES FOLDER Cmd)
