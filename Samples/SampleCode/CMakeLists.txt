PROJECT(vtku3dexporter)
CMAKE_MINIMUM_REQUIRED(VERSION 3.5)

SET(CMAKE_CXX_STANDARD 11)
SET(CMAKE_CXX_STANDARD_REQUIRED ON)

OPTION(WRAP_PYTHON "Build Python wrappers" ON)

# Use INSTALL_PYTHON_MODULE_DIR to manipulate where your lib and
# bin files will go. Useful for building wheels.
IF(NOT INSTALL_PYTHON_MODULE_DIR)
    SET(INSTALL_PYTHON_MODULE_DIR ".")
ENDIF()

IF(NOT LIB_DESTINATION)
    SET(LIB_DESTINATION "${INSTALL_PYTHON_MODULE_DIR}")
ENDIF()
IF(NOT BIN_DESTINATION)
    SET(BIN_DESTINATION "${INSTALL_PYTHON_MODULE_DIR}/bin")
ENDIF()
IF(NOT INCLUDE_DESTINATION)
    SET(INCLUDE_DESTINATION "include/${PROJECT_NAME}")
ENDIF()

MESSAGE( STATUS "CMAKE_INSTALL_PREFIX:    " ${CMAKE_INSTALL_PREFIX} )
MESSAGE( STATUS "LIB_DESTINATION:         " ${LIB_DESTINATION} )

# Include the source directory to make sure that out-of-source
# builds can find the proper headers.
INCLUDE_DIRECTORIES(${CMAKE_CURRENT_SOURCE_DIR})

# Find U3D
FIND_LIBRARY(U3D_LIB NAMES IDTF)
IF(NOT U3D_LIB)
    MESSAGE(FATAL_ERROR "Could not find U3D library")
ENDIF()
MESSAGE(STATUS "Found U3D library at: ${U3D_LIB}")

FIND_PATH(U3D_INCLUDE_DIR u3d/SceneConverterLib.h)
IF(NOT U3D_INCLUDE_DIR)
    MESSAGE(FATAL_ERROR "Could not find U3D include dir")
ENDIF()
MESSAGE(STATUS "Found U3D include dir: ${U3D_INCLUDE_DIR}")
INCLUDE_DIRECTORIES(${U3D_INCLUDE_DIR}/u3d)

# Find VTK
FIND_PACKAGE(VTK REQUIRED
    vtkIOExport
    vtkRenderingCore
    vtkWrappingPythonCore)
MESSAGE(STATUS "Found VTK libraries: ${VTK_LIBRARIES}")
MESSAGE(STATUS "Found UseVTK.cmake: ${VTK_USE_FILE}")
INCLUDE(${VTK_USE_FILE})
SET(VTK_MODULES_USED vtkIOExport)

# Remove the unexisting libpython file from VTK_LIBRARIES
# Otherwise, build will fail
LIST(FILTER VTK_LIBRARIES EXCLUDE REGEX libpython|\.lib)
MESSAGE(STATUS "Filtered VTK libraries: ${VTK_LIBRARIES}")

# Create vtkU3DExporter shared library
SET(LIBS ${VTK_LIBRARIES} ${U3D_LIB})
SET(EXPORTER_SRC vtkU3DExporter.cxx vtkU3DExporter.h)
ADD_LIBRARY(${PROJECT_NAME} SHARED ${EXPORTER_SRC})
TARGET_LINK_LIBRARIES(${PROJECT_NAME}
    ${LIBS})
IF(WIN32)
    TARGET_LINK_LIBRARIES(${PROJECT_NAME}
        winmm)
ENDIF(WIN32)
INSTALL(TARGETS ${PROJECT_NAME}
    RUNTIME DESTINATION ${BIN_DESTINATION}
    ARCHIVE DESTINATION ${LIB_DESTINATION}
    LIBRARY DESTINATION ${LIB_DESTINATION}
)
INSTALL(DIRECTORY .
    DESTINATION ${INCLUDE_DESTINATION}
    FILES_MATCHING PATTERN "*.h"
    PATTERN "CMake" EXCLUDE
)
MESSAGE(STATUS "Added library: " "${PROJECT_NAME}")

if(WRAP_PYTHON)
    # Find Python
    FIND_PACKAGE(PythonLibs REQUIRED)
    MESSAGE(STATUS "Found Python include dir: " ${PYTHON_INCLUDE_DIRS} )
    MESSAGE(STATUS "Found Python library: " ${PYTHON_LIBRARIES} )
    INCLUDE_DIRECTORIES("${PYTHON_INCLUDE_DIRS}")

    # Generate VTK Python wrappers
    INCLUDE(vtkWrapPython)
    # despite the name, this works for any Python version (tested up to 3.5)
    VTK_WRAP_PYTHON3(${PROJECT_NAME}Python PYTHON_SRCS "${EXPORTER_SRC}")
    MESSAGE(STATUS "Generated Python wrappers: " "${PYTHON_SRCS}")

    # Create PythonD library (brings everything together)
    ADD_LIBRARY("${PROJECT_NAME}PythonD" ${PYTHON_SRCS} "${EXPORTER_SRC}")
    SET_TARGET_PROPERTIES("${PROJECT_NAME}PythonD" PROPERTIES POSITION_INDEPENDENT_CODE ON)
    IF(WIN32)
        TARGET_LINK_LIBRARIES("${PROJECT_NAME}PythonD"
            winmm)
    ENDIF(WIN32)
    TARGET_LINK_LIBRARIES("${PROJECT_NAME}PythonD"
        ${LIBS}
        vtkWrappingPythonCore
        ${VTK_PYTHON_LIBRARIES})
    INSTALL(TARGETS "${PROJECT_NAME}PythonD"
        RUNTIME DESTINATION ${BIN_DESTINATION}
        ARCHIVE DESTINATION ${LIB_DESTINATION}
        LIBRARY DESTINATION ${LIB_DESTINATION}
    )

    # Create Python library (exposes to Python through a pyd)
    ADD_LIBRARY("${PROJECT_NAME}Python" MODULE "${PROJECT_NAME}PythonInit.cxx" "${PROJECT_NAME}PythonInitImpl.cxx")
    # collect the used pythonD libs in VTK_PYTHOND_LIBS
    SET(VTK_PYTHOND_LIBS)
    FOREACH(TMP_LIB ${VTK_MODULES_USED})
        SET(VTK_PYTHOND_LIBS ${VTK_PYTHOND_LIBS} ${TMP_LIB}PythonD)
    ENDFOREACH()

    # link 'em up
    # On linux, don't link the PYTHON_LIBRARIES
    IF(UNIX AND NOT APPLE)
        TARGET_LINK_LIBRARIES("${PROJECT_NAME}Python"
            "${PROJECT_NAME}PythonD"
            ${VTK_PYTHOND_LIBS})
    ELSE()
        TARGET_LINK_LIBRARIES("${PROJECT_NAME}Python"
            "${PROJECT_NAME}PythonD"
            ${PYTHON_LIBRARIES}
            ${VTK_PYTHOND_LIBS})
    ENDIF(UNIX AND NOT APPLE)
    # prettify the output
    SET_TARGET_PROPERTIES("${PROJECT_NAME}Python" PROPERTIES PREFIX "")
    if(WIN32 AND NOT CYGWIN)
        SET_TARGET_PROPERTIES("${PROJECT_NAME}Python" PROPERTIES SUFFIX ".pyd")
    endif(WIN32 AND NOT CYGWIN)
    # install command for the pyd
    INSTALL(TARGETS "${PROJECT_NAME}Python"
        LIBRARY DESTINATION "${INSTALL_PYTHON_MODULE_DIR}/${PROJECT_NAME}"
    )
    # install __init__.py to make it importable
    FILE(WRITE __init__.py "\
from __future__ import absolute_import\n\
# first importing vtk solves some import problems\n\
import vtk\n\
\n\
try:\n\
    # use relative import for installed modules\n\
    from .${PROJECT_NAME}Python import *\n\
except ImportError:\n\
    # during build and testing, the modules will be elsewhere,\n\
    # e.g. in lib directory or Release/Debug config directories\n\
    from ${PROJECT_NAME}Python import *\n\
")
    INSTALL(FILES __init__.py
        DESTINATION "${INSTALL_PYTHON_MODULE_DIR}/${PROJECT_NAME}"
    )


endif(WRAP_PYTHON)
