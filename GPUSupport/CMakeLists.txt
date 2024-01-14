# Copyright (c) Microsoft Corporation. All rights reserved.
# Licensed under the MIT License.

find_package(CUDA)

if (NOT GPU)
    set (CUDA_FOUND false)
endif()
#set(CUDA_FOUND false)

if (CUDA_FOUND)
    message (STATUS "Found cuda.")
    message (STATUS "Include Path:" ${CUDA_INCLUDE_DIRS})
    message (STATUS "Library Path:" ${CUDA_LIBRARIES})
    if(${CMAKE_CXX_COMPILER_ID} STREQUAL "GNU")
        set (CUDA_NVCC_FLAGS "${CUDA_NVCC_FLAGS} -Xcompiler -fPIC -Xcompiler -fopenmp -std=c++17 -Xptxas -O3 --use_fast_math --disable-warnings -lineinfo
        -gencode arch=compute_70,code=sm_70
        -gencode arch=compute_75,code=sm_75
        -gencode arch=compute_80,code=sm_80" )
    elseif(WIN32)
        set (CUDA_NVCC_FLAGS "${CUDA_NVCC_FLAGS} -Xcompiler /openmp -Xcompiler /std:c++17 -Xcompiler /Zc:__cplusplus -Xcompiler /FS --use_fast_math
        -gencode arch=compute_70,code=sm_70
        -gencode arch=compute_75,code=sm_75
        -gencode arch=compute_80,code=sm_80" )
    endif()

    message (STATUS "CUDA_NVCC_FLAGS:" ${CUDA_NVCC_FLAGS})
    # lxy 11.26 找不到spdk,在这添加
    set(AnnService ${PROJECT_SOURCE_DIR}/AnnService)
    set(Zstd ${PROJECT_SOURCE_DIR}/ThirdParty/zstd)
    set(Spdk ${PROJECT_SOURCE_DIR}/ThirdParty/spdk/build)
    set(Dpdk ${PROJECT_SOURCE_DIR}/ThirdParty/spdk/dpdk/build)
    set(IsalLCrypto ${PROJECT_SOURCE_DIR}/ThirdParty/isal-l_crypto/.libs/libisal_crypto.a)
    set(SpdkLibPrefix ${Spdk}/lib/libspdk_)
    set(DpdkLibPrefix ${Dpdk}/lib/librte_)
    set(SPDK_LIBRARIES -Wl,--whole-archive ${SpdkLibPrefix}bdev_nvme.a ${SpdkLibPrefix}bdev.a ${SpdkLibPrefix}nvme.a ${SpdkLibPrefix}vfio_user.a ${SpdkLibPrefix}sock.a ${SpdkLibPrefix}dma.a ${SpdkLibPrefix}notify.a ${SpdkLibPrefix}accel.a ${SpdkLibPrefix}event_bdev.a ${SpdkLibPrefix}event_accel.a ${SpdkLibPrefix}vmd.a ${SpdkLibPrefix}event_vmd.a ${SpdkLibPrefix}event_sock.a ${SpdkLibPrefix}event_iobuf.a ${SpdkLibPrefix}event.a ${SpdkLibPrefix}env_dpdk.a ${SpdkLibPrefix}log.a ${SpdkLibPrefix}thread.a ${SpdkLibPrefix}rpc.a ${SpdkLibPrefix}init.a ${SpdkLibPrefix}jsonrpc.a ${SpdkLibPrefix}json.a ${SpdkLibPrefix}trace.a ${SpdkLibPrefix}util.a ${DpdkLibPrefix}mempool.a ${DpdkLibPrefix}mempool_ring.a ${DpdkLibPrefix}eal.a ${DpdkLibPrefix}ring.a ${DpdkLibPrefix}telemetry.a ${DpdkLibPrefix}bus_pci.a ${DpdkLibPrefix}kvargs.a ${DpdkLibPrefix}pci.a -Wl,--no-whole-archive dl rt isal ${IsalLCrypto} uuid ${LIBBSD_LIBRARY})


    find_package(RocksDB CONFIG)
    if((DEFINED RocksDB_DIR) AND RocksDB_DIR)
        list(APPEND RocksDB_LIBRARIES RocksDB::rocksdb)
        find_package(uring)
        set(uring_LIBRARIES "")
        message (STATUS "GPU Found RocksDB ${RocksDB_VERSION}")
        message (STATUS "GPU RocksDB: ${RocksDB_DIR}")
        #message (STATUS "Include Path: ${RocksDB_INCLUDE_DIRS}")
    else()
        set(RocksDB_LIBRARIES "")
        set(uring_LIBRARIES "")
        message (FATAL_ERROR "Could not find RocksDB!")
    endif()
    # 找tbb
    find_package(TBB REQUIRED)
    if (TBB_FOUND)
        message (STATUS "Found TBB")
    endif()


    include_directories(${AnnService})

    # include_directories(${PROJECT_SOURCE_DIR}/ThirdParty/zstd/lib)   
    include_directories(${Zstd}/lib)
    include_directories(${Spdk}/include)
    file(GLOB_RECURSE GPU_HDR_FILES ${AnnService}/inc/Core/*.h  ${AnnService}/inc/Helper/*.h ${AnnService}/inc/Core/Common/cuda/*)
    file(GLOB_RECURSE GPU_SRC_FILES ${AnnService}/src/Core/*.cpp ${AnnService}/src/Helper/*.cpp ${AnnService}/src/Core/Common/Kernel.cu)

    list(REMOVE_ITEM GPU_HDR_FILES
        ${AnnService}/inc/Core/Common/DistanceUtils.h
        ${AnnService}/inc/Core/Common/InstructionUtils.h
        ${AnnService}/inc/Core/Common/CommonUtils.h
        ${AnnService}/inc/Core/Common/cuda/tests/
    )
    list(REMOVE_ITEM GPU_SRC_FILES
        ${AnnService}/src/Core/Common/DistanceUtils.cpp
        ${AnnService}/src/Core/Common/InstructionUtils.cpp
        ${AnnService}/src/Core/Common/cuda/tests/
    )

    set_source_files_properties(${GPU_SRC_FILES} PROPERTIES CUDA_SOURCE_PROPERTY_FORMAT OBJ)
    if (${CMAKE_BUILD_TYPE} STREQUAL "Debug")
        set(Definition -DGPU -DDEBUG)
    else()
        set(Definition -DGPU)
    endif()

    CUDA_ADD_LIBRARY(GPUSPTAGLib SHARED ${GPU_SRC_FILES} ${GPU_HDR_FILES})
    target_link_libraries(GPUSPTAGLib DistanceUtils ${RocksDB_LIBRARIES} ${uring_LIBRARIES} ${Boost_LIBRARIES} ${CUDA_LIBRARIES} libzstd_shared ${NUMA_LIBRARY} tbb ${SPDK_LIBRARIES} ${LIBBSD_LIBRARY} ${CUDA_LIBRARIES})

    # target_link_libraries(GPUSPTAGLib DistanceUtils ${RocksDB_LIBRARIES} ${uring_LIBRARIES} ${Boost_LIBRARIES} ${CUDA_LIBRARIES} libzstd_shared ${NUMA_LIBRARY} tbb ${SPDK_LIBRARIES} ${LIBBSD_LIBRARY})
    #target_link_libraries(GPUSPTAGLib DistanceUtils ${RocksDB_LIBRARIES} ${uring_LIBRARIES} ${Boost_LIBRARIES} ${CUDA_LIBRARIES} libzstd_shared ${NUMA_LIBRARY} tbb ${SPDK_LIBRARIES} ${LIBBSD_LIBRARY} ${CUDA_LIBRARIES})

    # target_link_libraries(GPUSPTAGLib DistanceUtils ${RocksDB_LIBRARIES} ${uring_LIBRARIES} ${Boost_LIBRARIES} ${CUDA_LIBRARIES} libzstd_shared ${NUMA_LIBRARY} tbb ${SPDK_LIBRARIES} ${LIBBSD_LIBRARY} ${CUDA_LIBRARIES})
    target_compile_definitions(GPUSPTAGLib PRIVATE ${Definition})

    CUDA_ADD_LIBRARY(GPUSPTAGLibStatic STATIC ${GPU_SRC_FILES} ${GPU_HDR_FILES})
    #target_link_libraries(GPUSPTAGLibStatic DistanceUtils ${RocksDB_LIBRARIES} ${uring_LIBRARIES} ${Boost_LIBRARIES} ${CUDA_LIBRARIES} libzstd_static ${NUMA_LIBRARY_STATIC} tbb ${SPDK_LIBRARIES} ${LIBBSD_LIBRARY})
    target_link_libraries(GPUSPTAGLibStatic DistanceUtils ${RocksDB_LIBRARIES} ${uring_LIBRARIES} ${Boost_LIBRARIES} libzstd_static ${NUMA_LIBRARY_STATIC} tbb ${SPDK_LIBRARIES} ${LIBBSD_LIBRARY} ${CUDA_LIBRARIES}) #

    target_compile_definitions(GPUSPTAGLibStatic PRIVATE ${Definition})
    add_dependencies(GPUSPTAGLibStatic GPUSPTAGLib)

    CUDA_ADD_EXECUTABLE(gpuindexbuilder ${AnnService}/src/IndexBuilder/main.cpp)
    # target_link_libraries(gpuindexbuilder GPUSPTAGLibStatic ${Boost_LIBRARIES} ${CUDA_LIBRARIES})
    target_link_libraries(gpuindexbuilder GPUSPTAGLibStatic ${Boost_LIBRARIES}) #

    target_compile_definitions(gpuindexbuilder PRIVATE ${Definition})

    set(VECTORSEARCH_INC_DIR ${AnnService}/inc/SSDServing/VectorSearch)
    set(VECTORSEARCH_IMP_DIR ${AnnService}/src/SSDServing/VectorSearch)
    file(GLOB_RECURSE SSD_SERVING_HDR_FILES ${AnnService}/inc/SSDServing/*.h)
    file(GLOB_RECURSE SSD_SERVING_FILES ${AnnService}/src/SSDServing/*.cpp)
    if(NOT WIN32)
        list(REMOVE_ITEM SSD_SERVING_HDR_FILES 
            ${VECTORSEARCH_INC_DIR}/AsyncFileReader.h
            )
    elseif(WIN32)
        list(REMOVE_ITEM SSD_SERVING_HDR_FILES
            ${VECTORSEARCH_INC_DIR}/AsyncFileReaderLinux.h
            )
    endif()

    CUDA_ADD_EXECUTABLE(gpussdserving ${SSD_SERVING_HDR_FILES} ${SSD_SERVING_FILES})
    # target_link_libraries(gpussdserving GPUSPTAGLibStatic ${Boost_LIBRARIES} ${CUDA_LIBRARIES})
    target_link_libraries(gpussdserving GPUSPTAGLibStatic ${Boost_LIBRARIES} )#

    target_compile_definitions(gpussdserving PRIVATE ${Definition} _exe)
else()
    message (STATUS "Could not find cuda.")
endif()
