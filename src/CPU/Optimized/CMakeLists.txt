project(cpu-optimized)

# sources and header files
set (${PROJECT_NAME}_headers
    Optimized.h
)

set (${PROJECT_NAME}_sources
    Optimized.cpp
)

# create library
add_library(
	${PROJECT_NAME} OBJECT ${${PROJECT_NAME}_headers}
	                       ${${PROJECT_NAME}_sources}
)
if (NOT BUILD_STATIC_LIBS)
set_target_properties(${PROJECT_NAME} PROPERTIES COMPILE_FLAGS "-fPIC")
endif()

# Copy header files
install(
    FILES
    ${${PROJECT_NAME}_headers}
    DESTINATION
    include/CPU/Optimized
)

# Add kernels
add_subdirectory(kernels)
