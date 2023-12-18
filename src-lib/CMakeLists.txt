# Darknet object detection framework


SET (CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON) # automatically create a module definition file with all global symbols
CONFIGURE_FILE (${CMAKE_CURRENT_SOURCE_DIR}/darknet_version.h.in ${CMAKE_CURRENT_SOURCE_DIR}/darknet_version.h)


# ==
# The library is somewhat more complicated than the original CLI.  It has lots of source files, more optional files
# depending on whether or not CUDA is enabled, and yet more files if we're building on Windows.
# ==
FILE (GLOB LIBSRC *.cpp)
LIST (REMOVE_ITEM LIBSRC yolo_v2_class.cpp)

IF (DARKNET_USE_CUDA)
	FILE (GLOB CUDASRC *.cu)
	LIST (APPEND LIBSRC ${CUDASRC})
ENDIF ()

IF (WIN32)
	INCLUDE_DIRECTORIES (windows)
	FILE (GLOB WINSRC windows/*.c)
	LIST (APPEND LIBSRC ${WINSRC})
ENDIF ()

LIST (SORT LIBSRC)


# ==
# First thing we do is build an object library so things don't have to be compiled multiple times.
# ==
MESSAGE(STATUS "Setting up DARKNET OBJ")
ADD_LIBRARY(darknetobjlib OBJECT ${LIBSRC})
IF (DARKNET_USE_CUDA)
	SET_TARGET_PROPERTIES (darknetobjlib PROPERTIES CUDA_ARCHITECTURES "${DARKNET_CUDA_ARCHITECTURES}")
	SET_TARGET_PROPERTIES (darknetobjlib PROPERTIES CUDA_SEPARABLE_COMPILATION OFF)
	SET_TARGET_PROPERTIES (darknetobjlib PROPERTIES CUDA_RESOLVE_DEVICE_SYMBOLS OFF)
ENDIF ()


# ==
# Next step is to create the actual darknet library.
# ==
MESSAGE(STATUS "Setting up DARKNET LIB")
ADD_LIBRARY (darknet SHARED $<TARGET_OBJECTS:darknetobjlib>)
SET_TARGET_PROPERTIES (darknet PROPERTIES PUBLIC_HEADER "darknet.h;darknet_version.h")
IF (DARKNET_USE_CUDA)
	SET_TARGET_PROPERTIES (darknet PROPERTIES CUDA_ARCHITECTURES "${DARKNET_CUDA_ARCHITECTURES}")
	SET_TARGET_PROPERTIES (darknet PROPERTIES CUDA_SEPARABLE_COMPILATION OFF)
	SET_TARGET_PROPERTIES (darknet PROPERTIES CUDA_RESOLVE_DEVICE_SYMBOLS OFF)
ENDIF ()
TARGET_LINK_LIBRARIES (darknet PRIVATE ${DARKNET_LINK_LIBS})
INSTALL (TARGETS darknet LIBRARY DESTINATION lib PUBLIC_HEADER DESTINATION include)


# ==
# The old C++ YOLOv2 library is still used by some projects.  E.g., the original C# wrapper.
# But as far as I can tell, this library was only ever used on Windows.  Doesn't look like
# the old Linux Makefile ever tried to build this on Linux.
# ==
IF (WIN32)
	MESSAGE(STATUS "Setting up YOLO V2 LIB")
	ADD_LIBRARY (yolo_v2_class SHARED yolo_v2_class.cpp)
	SET_TARGET_PROPERTIES (yolo_v2_class PROPERTIES PUBLIC_HEADER "yolo_v2_class.hpp")
	TARGET_LINK_LIBRARIES (yolo_v2_class PRIVATE ${DARKNET_LINK_LIBS} darknet)
	INSTALL (TARGETS yolo_v2_class LIBRARY DESTINATION lib PUBLIC_HEADER DESTINATION include)
ENDIF ()


# The CLI used to be called "darknet" instead of "darknetcli".  But this conflicted with "darknet" the library target.
# It caused strange problems on Windows where the DLL would depend on the .exe file.  To solve this we ended up
# renaming the CLI from "darknet" to "darknetcli".  However, people with a build environment from before this change
# was made (2023-12-17) might still have a copy of the _old_ CLI in their build output directory.  If this is the case,
# to prevent anyone from accidentally using this old CLI, we're going to rename it.
#GET_TARGET_PROPERTY(CLI_OUTPUT_DIR darknetcli BINARY_DIR)
#IF (WIN32)
#	SET (OLD_CLI "${CLI_OUTPUT_DIR}/${CMAKE_BUILD_TYPE}/darknet.exe")
#ELSE ()
#	SET (OLD_CLI "${CLI_OUTPUT_DIR}/darknet")
#ENDIF ()
#IF (EXISTS "${OLD_CLI}")
#	MESSAGE (WARNING "An old copy of the CLI was found and will be renamed: ${OLD_CLI}")
#	FILE (RENAME "${OLD_CLI}" "${OLD_CLI}_OLD")
#ENDIF ()