add_subdirectory(detail)

function(math_library TARGET)
    # math_library is a function to create math library.
    # The interface is the same as cc_library.
    # But it handle split GPU/CPU code and link some common library.
    set(cc_srcs)
    set(cu_srcs)
    set(hip_srcs)
    set(math_common_deps device_context framework_proto enforce)
    if (WITH_GPU)
        if (${CMAKE_CUDA_COMPILER_VERSION} LESS 11.0)	
            list(APPEND math_common_deps cub)
	else()
            list(APPEND math_common_deps)
	endif()
    endif()
    set(multiValueArgs DEPS)
    cmake_parse_arguments(math_library "${options}" "${oneValueArgs}"
            "${multiValueArgs}" ${ARGN})

    if (EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/${TARGET}.cc)
        list(APPEND cc_srcs ${TARGET}.cc)
    endif()
    if (EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/${TARGET}.cu)
        list(APPEND cu_srcs ${TARGET}.cu)
    endif()
    if (EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/${TARGET}.cu.cc)
        list(APPEND cu_srcs ${TARGET}.cu.cc)
    endif()

    list(LENGTH cc_srcs cc_srcs_len)
    if (WITH_GPU)
        nv_library(${TARGET} SRCS ${cc_srcs} ${cu_srcs} DEPS ${math_library_DEPS} ${math_common_deps})
    elseif (WITH_ROCM)
        hip_library(${TARGET} SRCS ${cc_srcs} ${cu_srcs} DEPS ${math_library_DEPS} ${math_common_deps})
    elseif(${cc_srcs_len} GREATER 0)
        cc_library(${TARGET} SRCS ${cc_srcs} DEPS ${math_library_DEPS} ${math_common_deps})
    endif()
endfunction()

if (WITH_ASCEND_CL)
  cc_library(beam_search_npu SRCS beam_search_npu.cc DEPS npu_op_runner)
endif()

# please add new math_library in alphabetical order
math_library(concat_and_split)
math_library(context_project DEPS im2col math_function)
math_library(cross_entropy)
math_library(cos_sim_functor)
math_library(depthwise_conv)
math_library(im2col)
math_library(sample_prob)
math_library(sampler DEPS generator)

math_library(gru_compute DEPS activation_functions math_function)
math_library(lstm_compute DEPS activation_functions)

cc_library(blas SRCS blas.cc DEPS cblas framework_proto device_context)
math_library(math_function DEPS blas)
math_library(maxouting)
math_library(pooling)

if(WITH_MKLDNN)
    math_library(selected_rows_functor DEPS selected_rows math_function blas mkldnn_axpy_handler)
else()
    math_library(selected_rows_functor DEPS selected_rows math_function blas)
endif()

math_library(sequence2batch)
math_library(sequence_padding)
math_library(sequence_pooling DEPS math_function jit_kernel_helper)
math_library(sequence_scale)
math_library(softmax DEPS math_function jit_kernel_helper)
if (WITH_ASCEND_CL)
    math_library(beam_search DEPS math_function beam_search_npu)
else()
    math_library(beam_search DEPS math_function)
endif()
math_library(fc DEPS blas)
math_library(lapack_function DEPS dynload_lapack)

math_library(matrix_bit_code)

math_library(unpooling)
math_library(vol2col)
math_library(prelu)
math_library(bert_encoder_functor)
math_library(tree2col DEPS math_function)
math_library(matrix_inverse)
math_library(segment_pooling)
math_library(matrix_solve)

cc_test(math_function_test SRCS math_function_test.cc DEPS math_function)
cc_test(selected_rows_functor_test SRCS selected_rows_functor_test.cc DEPS selected_rows_functor)
cc_test(im2col_test SRCS im2col_test.cc DEPS im2col)
cc_test(vol2col_test SRCS vol2col_test.cc DEPS vol2col)
cc_test(sequence_padding_test SRCS sequence_padding_test.cc DEPS sequence_padding)
cc_test(sequence_pooling_test SRCS sequence_pooling_test.cc DEPS sequence_pooling)
cc_test(beam_search_test SRCS beam_search_test.cc DEPS beam_search)
if(WITH_GPU)
    nv_test(math_function_gpu_test SRCS math_function_test.cu DEPS math_function)
    nv_test(selected_rows_functor_gpu_test SRCS selected_rows_functor_test.cu.cc DEPS selected_rows_functor math_function)
endif()
if(WITH_ROCM)
    hip_test(math_function_gpu_test SRCS math_function_test.cu DEPS math_function tensor)
    hip_test(selected_rows_functor_gpu_test SRCS selected_rows_functor_test.cu.cc DEPS selected_rows_functor math_function)
endif()
cc_test(concat_test SRCS concat_test.cc DEPS concat_and_split)
cc_test(cpu_vec_test SRCS cpu_vec_test.cc DEPS blas cpu_info)
if(WITH_TESTING AND TEST im2col_test)
    set_tests_properties(im2col_test PROPERTIES TIMEOUT 120)
endif()
