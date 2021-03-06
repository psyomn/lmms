INCLUDE(BuildPlugin)

SET(ZYN_SRC_GUI
	"${CMAKE_CURRENT_SOURCE_DIR}/src/UI/MasterUI.cc"
	"${CMAKE_CURRENT_SOURCE_DIR}/src/UI/VirKeyboard.cc"
	"${CMAKE_CURRENT_SOURCE_DIR}/src/UI/ConfigUI.cc"
	"${CMAKE_CURRENT_SOURCE_DIR}/src/UI/OscilGenUI.cc"
	"${CMAKE_CURRENT_SOURCE_DIR}/src/UI/WidgetPDial.cc"
	"${CMAKE_CURRENT_SOURCE_DIR}/src/UI/ResonanceUI.cc"
	"${CMAKE_CURRENT_SOURCE_DIR}/src/UI/MicrotonalUI.cc"
	"${CMAKE_CURRENT_SOURCE_DIR}/src/UI/ADnoteUI.cc"
	"${CMAKE_CURRENT_SOURCE_DIR}/src/UI/PresetsUI.cc"
	"${CMAKE_CURRENT_SOURCE_DIR}/src/UI/EffUI.cc"
	"${CMAKE_CURRENT_SOURCE_DIR}/src/UI/FilterUI.cc"
	"${CMAKE_CURRENT_SOURCE_DIR}/src/UI/PartUI.cc"
	"${CMAKE_CURRENT_SOURCE_DIR}/src/UI/EnvelopeUI.cc"
	"${CMAKE_CURRENT_SOURCE_DIR}/src/UI/LFOUI.cc"
	"${CMAKE_CURRENT_SOURCE_DIR}/src/UI/SeqUI.cc"
	"${CMAKE_CURRENT_SOURCE_DIR}/src/UI/PADnoteUI.cc"
	"${CMAKE_CURRENT_SOURCE_DIR}/src/UI/SUBnoteUI.cc"
	"${CMAKE_CURRENT_SOURCE_DIR}/src/UI/BankUI.cc"
)


# definitions for ZynAddSubFX
IF(LMMS_BUILD_LINUX OR LMMS_BUILD_APPLE)
	ADD_DEFINITIONS(-DOS_LINUX)
ELSE(LMMS_BUILD_LINUX OR LMMS_BUILD_APPLE)
	ADD_DEFINITIONS(-DOS_WINDOWS)
ENDIF(LMMS_BUILD_LINUX OR LMMS_BUILD_APPLE)

# do not conflict with LMMS' Controller class
ADD_DEFINITIONS(-DController=ZynController)

# use asm optimizations when on x86 or x86_64
IF(LMMS_HOST_X86 OR LMMS_HOST_X86_64)
	ADD_DEFINITIONS(-DASM_F2I_YES)
ENDIF(LMMS_HOST_X86 OR LMMS_HOST_X86_64)

# build ZynAddSubFX with full optimizations
SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O2 -Wno-write-strings -Wno-deprecated-declarations")

# link system-libraries when on win32
IF(LMMS_BUILD_WIN32)
	ADD_DEFINITIONS(-DPTW32_STATIC_LIB)
ENDIF(LMMS_BUILD_WIN32)

SET(FLTK_SKIP_OPENGL TRUE)
SET(FLTK_SKIP_FORMS TRUE)
SET(FLTK_SKIP_IMAGES TRUE)
SET(FLTK_SKIP_FLUID TRUE)

FIND_PACKAGE(FLTK REQUIRED)

INCLUDE_DIRECTORIES("${FLTK_INCLUDE_DIR}"
                    "${CMAKE_CURRENT_SOURCE_DIR}"
                    ${FFTW3F_INCLUDE_DIRS}
                    "${CMAKE_CURRENT_BINARY_DIR}"
                    "${CMAKE_BINARY_DIR}")
set(ZASF_CORE_LIBS
    zynaddsubfx_input
    zynaddsubfx_output
    zynaddsubfx_misc
    zynaddsubfx_synth
    zynaddsubfx_seq
    zynaddsubfx_effect
    zynaddsubfx_params
    zynaddsubfx_dsp
    zynaddsubfx_samples
    )

macro(unit_test NAME CXX_FILE FILES)
endmacro(unit_test)

add_subdirectory(src/Misc)
add_subdirectory(src/Input)
add_subdirectory(src/Synth)
add_subdirectory(src/Output)
add_subdirectory(src/Seq)
add_subdirectory(src/Effects)
add_subdirectory(src/Params)
add_subdirectory(src/DSP)
add_subdirectory(src/Samples)

ADD_LIBRARY(ZynAddSubFxCore SHARED LocalZynAddSubFx.cpp)
TARGET_LINK_LIBRARIES(ZynAddSubFxCore ${ZASF_CORE_LIBS} ${FFTW3F_LIBRARIES} ${QT_LIBRARIES} -lz -lpthread)
IF(LMMS_BUILD_WIN32)
	TARGET_LINK_LIBRARIES(ZynAddSubFxCore -lws2_32)
	INSTALL(TARGETS ZynAddSubFxCore RUNTIME DESTINATION "${PLUGIN_DIR}")
ELSE(LMMS_BUILD_WIN32)
	INSTALL(TARGETS ZynAddSubFxCore LIBRARY DESTINATION "${PLUGIN_DIR}")
ENDIF(LMMS_BUILD_WIN32)

BUILD_PLUGIN(zynaddsubfx ZynAddSubFx.cpp ZynAddSubFx.h MOCFILES ZynAddSubFx.h EMBEDDED_RESOURCES "${CMAKE_CURRENT_SOURCE_DIR}/*.png")
TARGET_LINK_LIBRARIES(zynaddsubfx ZynAddSubFxCore)

IF(WIN32)
	SET(WINRC "${CMAKE_CURRENT_BINARY_DIR}/zynaddsubfxrc.obj")
	ADD_CUSTOM_COMMAND(OUTPUT "${WINRC}"
				COMMAND "${WINDRES}"
					"-I\"${CMAKE_CURRENT_SOURCE_DIR}\""
					"-o\"${CMAKE_CURRENT_BINARY_DIR}/zynaddsubfxrc.obj\""
					"-i\"${CMAKE_CURRENT_BINARY_DIR}/zynaddsubfx.rc\""
				DEPENDS "${CMAKE_CURRENT_BINARY_DIR}/zynaddsubfx.rc")
ENDIF(WIN32)

SET(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/${PLUGIN_DIR}")
SET(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)
ADD_EXECUTABLE(RemoteZynAddSubFx RemoteZynAddSubFx.cpp ${ZYN_SRC_GUI} "${WINRC}")
INSTALL(TARGETS RemoteZynAddSubFx RUNTIME DESTINATION "${PLUGIN_DIR}")
TARGET_LINK_LIBRARIES(RemoteZynAddSubFx ${FLTK_LIBRARIES} -lpthread "-L\"${CMAKE_CURRENT_BINARY_DIR}\"" -lZynAddSubFxCore )
ADD_DEPENDENCIES(RemoteZynAddSubFx ZynAddSubFxCore)

# link system libraries when on win32
IF(LMMS_BUILD_WIN32)
	TARGET_LINK_LIBRARIES(RemoteZynAddSubFx ${QT_LIBRARIES})
ENDIF(LMMS_BUILD_WIN32)

# FLTK needs X
IF(LMMS_BUILD_LINUX)
	TARGET_LINK_LIBRARIES(RemoteZynAddSubFx -ldl)
ENDIF(LMMS_BUILD_LINUX)


IF(LMMS_BUILD_WIN32)
	ADD_CUSTOM_COMMAND(TARGET ZynAddSubFxCore POST_BUILD COMMAND "${STRIP}" "\"${CMAKE_CURRENT_BINARY_DIR}/libZynAddSubFxCore.dll\"")
	ADD_CUSTOM_COMMAND(TARGET RemoteZynAddSubFx POST_BUILD COMMAND "${STRIP}" "\"${CMAKE_CURRENT_BINARY_DIR}/RemoteZynAddSubFx.exe\"")
ENDIF(LMMS_BUILD_WIN32)
