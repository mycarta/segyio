project(segyio-python)

if (REQUIRE_PYTHON)
    set(BUILD_PYTHON ON)
endif()

if (NOT BUILD_PYTHON)
    return()
endif()

find_package(PythonInterp REQUIRED)

if (NOT PYTHON_EXECUTABLE AND REQUIRE_PYTHON)
    message(SEND_ERROR "Could not find python executable")
    return()
endif()

if (NOT PYTHON_EXECUTABLE)
    message(WARNING "Could not find python - skipping python bindings")
    return()
endif()

if (PYTHON_INSTALL_LAYOUT)
    set(setup-install-layout --install-layout ${PYTHON_INSTALL_LAYOUT})
endif()

set(python ${PYTHON_EXECUTABLE})

if (NOT WIN32)
    # setuptools on microsoft compilers doesn't support the --library-dir or
    # --build-dir flag and crashes, so only pass it on non-microsoft platforms
    set(setup-py-libdir build_ext
                            --rpath        $<TARGET_FILE_DIR:segyio>
                            --library-dirs $<TARGET_FILE_DIR:segyio>)

    set(install-no-rpath install_lib --build-dir build/install)
    set(build-no-rpath --library-dirs $<TARGET_FILE_DIR:segyio>
                       build --build-lib build/install)
else ()
    set(copy-dll-to-src ${CMAKE_COMMAND} -E
        copy $<TARGET_FILE:segyio>
        ${CMAKE_CURRENT_SOURCE_DIR}/segyio/$<TARGET_FILE_NAME:segyio>)
endif ()


set(setup-py ${CMAKE_SOURCE_DIR}/setup.py)
add_custom_target(
    segyio-python ALL
    COMMENT "Building python library with setup.py"
    SOURCES ${setup-py}
    DEPENDS ${setup-py}
    VERBATIM

    # copy the examples to have them runnable from a relative directory, so
    # that a locally-installed segyio can be imported
    COMMAND ${CMAKE_COMMAND} -E
        copy_directory ${CMAKE_CURRENT_SOURCE_DIR}/examples .

    # do the same to tests. running through setup.py test is *very* slow, so
    # invoke unittest manually from the build directory. more importantly,
    # setup.py test will pollute the source directory with egg info and
    # extensions, which is unacceptable in a cmake world
    COMMAND ${CMAKE_COMMAND} -E
        copy_directory ${CMAKE_CURRENT_SOURCE_DIR}/test test

    # setuptools on windows breaks spectacularly when the library isn't
    # available in the same directory, and build_ext --library-dirs is not
    # support on msvc is not supported, so we must copy out the libsegyio core
    # object and put it here
    COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_FILE:segyio>
                                     $<TARGET_FILE_NAME:segyio>

    COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_LINKER_FILE:segyio>
                                     $<TARGET_LINKER_FILE_NAME:segyio>

    # on windows, copy the freshly-built dll to the source directory. this
    # voilates the cmake spirit (as does the version.py writing from
    # setuptools-scm), but there's no auditwheel like tool to help fix the
    # wheel, and the dll must still be bundled in order to make the package
    # work. it's paired with package_data in setup.py. this is necessary
    # because setup.py assumes all files to bundled with the package are
    # relative downwards and in the package itself, with poor support for
    # grabbing other files and adding to it later.
    COMMAND ${copy-dll-to-src}

    # install the lib in the build-dir so that the examples can load that from
    # current working dir
    COMMAND ${python} ${setup-py} ${setup-py-libdir} install_lib -d .

    # to maintain good make && make install behaviour, the extension is built
    # twice, one with rpath (for testing and build-dir-local) and one for
    # installation
    COMMAND ${python} ${setup-py} build_ext ${build-no-rpath}
)

add_dependencies(segyio-python segyio)

# write egg_info to the build dir in order not to pollute the source directory
# and install as if it was through some other distro by using single-version,
# so that install won't die on possibly missing pythonpath
# setup.py install doesn't respect DESTDIR, so "force" it by changing the
# --root if DESTDIR is passed
install(CODE "
if (DEFINED ENV{DESTDIR})
    get_filename_component(abs-destdir \"\$ENV{DESTDIR}\" ABSOLUTE)
    set(root_destdir --root=\${abs-destdir})
endif()

execute_process(COMMAND
${python} ${setup-py}
    install_egg_info --install-dir .
    egg_info --egg-base .
    ${install-no-rpath}
    install --prefix=${CMAKE_INSTALL_PREFIX}
            --single-version-externally-managed
            --record installed-files
            ${setup-install-layout}
            \${root_destdir}
WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
)")

option(BUILD_PYDOC "Build python documentation" OFF)

if(BUILD_DOC)
    set(BUILD_PYDOC ON)
endif()

if(BUILD_PYDOC)
    find_program(sphinx sphinx-build)

    if(NOT sphinx)
        message(WARNING "Could not find sphinx, skipping python documentation")
        set(BUILD_PYDOC OFF)
    endif()

endif()

if(BUILD_PYDOC AND sphinx)
    # use the -d argument to avoid putting cache dir in docs/, because that
    # directory will be install'd
    add_custom_target(pydoc
        COMMAND ${sphinx}
            -d ${CMAKE_CURRENT_BINARY_DIR}/.doctrees
            ${SPHINX_ARGS}
            ${CMAKE_CURRENT_SOURCE_DIR}/docs
            ${CMAKE_CURRENT_BINARY_DIR}/docs
        DEPENDS docs/conf.py
                docs/index.rst
                docs/segyio.rst
        COMMENT "Building python documentation with sphinx"
    )
    add_dependencies(doc pydoc)

    install(
        DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/docs/
        DESTINATION ${CMAKE_INSTALL_DATADIR}/doc/segyio
    )
endif()

if(NOT BUILD_TESTING)
    return()
endif()

file(GLOB sgys ${testdata}/*.sgy ${testdata}/*.su)
foreach (sgy ${sgys})
    get_filename_component(fl ${sgy} NAME)
    configure_file(${sgy} test-data/${fl} COPYONLY)
endforeach ()

set(pytest ${python} -m pytest)
add_test(NAME python.unit COMMAND ${pytest} test/)

configure_file(${testdata}/small.sgy
               test-data/write.sgy
               COPYONLY)

add_test(NAME python.example.about        COMMAND ${python} about.py test-data/small.sgy INLINE_3D CROSSLINE_3D)
add_test(NAME python.example.write        COMMAND ${python} write.py test-data/write.sgy)
add_test(NAME python.example.makefile     COMMAND ${python} make-file.py test-data/large-file.sgy 20 1 20 1 20)
add_test(NAME python.example.makepsfile   COMMAND ${python} make-ps-file.py test-data/small-prestack.sgy 10 1 5 1 4 1 3)
add_test(NAME python.example.subcube      COMMAND ${python} copy-sub-cube.py test-data/small.sgy test-data/copy.sgy)
add_test(NAME python.example.rotate       COMMAND ${python} make-rotated-copies.py test-data/small.sgy ex-rotate.sgy)
add_test(NAME python.example.scan_min_max COMMAND ${python} scan_min_max.py test-data/small.sgy)
add_test(NAME python.example.multi-text   COMMAND ${python} make-multiple-text.py test-data/multi-text.sgy)
