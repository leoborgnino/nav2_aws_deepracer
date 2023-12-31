include(PythonBindingsUtils)
if(PY_PYPLUSPLUS AND PY_PYGCCXML AND CASTXML_FOUND)
    option(OMPL_BUILD_PYBINDINGS "Build OMPL Python bindings" ON)
    option(OMPL_BUILD_PYTESTS "Build/run OMPL Python tests" ON)
else()
    option(OMPL_BUILD_PYBINDINGS "Build OMPL Python bindings" OFF)
    option(OMPL_BUILD_PYTESTS "Build/run OMPL Python tests" OFF)
endif()
add_feature_info(OMPL_BUILD_PYBINDINGS "${OMPL_BUILD_PYBINDINGS}" "Whether to build the Python bindings.")
add_feature_info(OMPL_BUILD_PYTESTS "${OMPL_BUILD_PYTESTS}" "Whether to build/run the python binding unit tests.")

if (OMPL_BUILD_PYBINDINGS)
    if(WIN32)
        # omit StateStorage class bindings on Windows;
        # Boost.Serialization does nasty things that break the bindings.
        set(PYOMPL_EXTRA_CFLAGS "${PYOMPL_EXTRA_CFLAGS} -DOMPL_BASE_STATE_STORAGE_ -DOMPL_BASE_PLANNER_DATA_STORAGE_ -DOMPL_CONTROL_PLANNER_DATA_STORAGE_")
        add_definitions(-DOMPL_BASE_STATE_STORAGE_ -DOMPL_BASE_PLANNER_DATA_STORAGE_ -DOMPL_CONTROL_PLANNER_DATA_STORAGE_)
    endif(WIN32)

    set(OMPL_MODULES util base control geometric tools)

    if(PY_OMPL_GENERATE)
        # need to run cmake to regenerate ompl/bindings_generator.py if
        # bindings_generator.py.in is changed.
        configure_file("${CMAKE_CURRENT_SOURCE_DIR}/bindings_generator.py.in"
            "${CMAKE_CURRENT_SOURCE_DIR}/ompl/bindings_generator.py" @ONLY)

        create_module_generation_targets(util)
        if(OMPL_HAVE_NUMPY)
            create_module_generation_targets(base)
        else()
            create_module_generation_targets(base "[Cc]onstrain")
        endif()
        add_dependencies(update_base_bindings update_util_bindings)
        create_module_generation_targets(geometric)
        add_dependencies(update_geometric_bindings update_base_bindings)
        create_module_generation_targets(control)
        add_dependencies(update_control_bindings update_geometric_bindings)
        add_dependencies(update_control_bindings update_base_bindings)
        create_module_generation_targets(tools)
        add_dependencies(update_tools_bindings update_control_bindings)
    endif(PY_OMPL_GENERATE)

    if(PY_OMPL_COMPILE AND EXISTS "${CMAKE_CURRENT_BINARY_DIR}/bindings")
        foreach(module ${OMPL_MODULES})
            create_module_target(${module})
        endforeach(module)
        if(OMPL_HAVE_NUMPY AND TARGET py_ompl_base)
            target_link_libraries(py_ompl_base ${Boost_NUMPY_LIBRARY})
        endif()

        install(DIRECTORY ompl DESTINATION "${OMPL_PYTHON_INSTALL_DIR}"
            COMPONENT python
            PATTERN "*.pyc" EXCLUDE
            PATTERN "__pycache__" EXCLUDE
            PATTERN "_*.so" EXCLUDE
            PATTERN "app" EXCLUDE)
        # put app submodule in omplapp component
        if(PROJECT_NAME STREQUAL "omplapp")
            install(DIRECTORY ompl/app
                DESTINATION "${OMPL_PYTHON_INSTALL_DIR}/ompl"
                COMPONENT omplapp
                PATTERN "*.pyc" EXCLUDE
                PATTERN "__pycache__" EXCLUDE
                PATTERN "_*.so" EXCLUDE)
        endif()
    endif()

endif(OMPL_BUILD_PYBINDINGS)

add_custom_target(clean_bindings
    "${CMAKE_COMMAND}" -E remove_directory "${CMAKE_CURRENT_BINARY_DIR}/bindings"
    COMMAND "${CMAKE_COMMAND}" -E remove -f pyplusplus_{base,control,geometric,tools,util}.{cache,log}
    WORKING_DIRECTORY "${PROJECT_BINARY_DIR}")
