cmake_minimum_required(VERSION 3.13.1)

set(MODEL_FOLDER "${CMAKE_CURRENT_SOURCE_DIR}/..")
set(EI_SDK_FOLDER "${CMAKE_CURRENT_SOURCE_DIR}/../edge-impulse-sdk")
set(EI_PLATFORM_FOLDER "${CMAKE_CURRENT_SOURCE_DIR}/../edge-impulse")
set(FIRMWARE_SDK_FOLDER "${CMAKE_CURRENT_SOURCE_DIR}/../firmware-sdk")
set(COMPONENTS_FOLDER "${CMAKE_CURRENT_SOURCE_DIR}/../components")

if(NOT CMAKE_BUILD_EARLY_EXPANSION)
    add_definitions(-DEI_CLASSIFIER_TFLITE_ENABLE_ESP_NN=1)
    add_definitions(-DEI_CLASSIFIER_TFLITE_ENABLE_ESP_NN_S3=1)
    add_definitions(-DEIDSP_USE_ESP_DSP=1)
endif()

set(include_dirs
    ${MODEL_FOLDER}
    ${MODEL_FOLDER}/tflite-model
    ${MODEL_FOLDER}/model-parameters
    ${EI_SDK_FOLDER}
    ${EI_SDK_FOLDER}/classifier
    ${EI_SDK_FOLDER}/dsp
    ${EI_SDK_FOLDER}/porting
    ${EI_SDK_FOLDER}/porting/espressif/ESP-NN
    ${EI_SDK_FOLDER}/porting/espressif/ESP-NN/include
    ${EI_SDK_FOLDER}/porting/espressif/esp-dsp/modules/fft/include
    ${EI_SDK_FOLDER}/porting/espressif/esp-dsp/modules/common/include
    ${EI_SDK_FOLDER}/tensorflow
    ${EI_SDK_FOLDER}/third_party
    ${EI_SDK_FOLDER}/third_party/ruy
    ${EI_SDK_FOLDER}/third_party/gemmlowp
    ${EI_SDK_FOLDER}/third_party/flatbuffers/include
    ${EI_PLATFORM_FOLDER}
    ${EI_PLATFORM_FOLDER}/ingestion-sdk-c
    ${EI_PLATFORM_FOLDER}/ingestion-sdk-platform/espressif_esp32
    ${EI_PLATFORM_FOLDER}/ingestion-sdk-platform/sensors
    ${EI_PLATFORM_FOLDER}/inference
    ${FIRMWARE_SDK_FOLDER}
    ${FIRMWARE_SDK_FOLDER}/at-server
    ${COMPONENTS_FOLDER}/LIS3DHTR_ESP-IDF
    ${COMPONENTS_FOLDER}/LIS3DHTR_ESP-IDF/src
    ${COMPONENTS_FOLDER}/LIS3DHTR_ESP-IDF/src/include
)

include(${EI_SDK_FOLDER}/cmake/utils.cmake)

RECURSIVE_FIND_FILE_EXCLUDE_DIR(SOURCE_FILES "${EI_SDK_FOLDER}" "ESP-NN" "*.cpp")
RECURSIVE_FIND_FILE_EXCLUDE_DIR(CC_FILES "${EI_SDK_FOLDER}" "ESP-NN" "*.cc")
RECURSIVE_FIND_FILE_EXCLUDE_DIR(C_FILES "${EI_SDK_FOLDER}" "ESP-NN" "*.c")

list(APPEND C_FILES "${EI_SDK_FOLDER}/CMSIS/DSP/Source/StatisticsFunctions/arm_rms_f32.c")

RECURSIVE_FIND_FILE(PORTING_FILES "${EI_SDK_FOLDER}/porting/espressif" "*.cpp")
RECURSIVE_FIND_FILE(FIRMWARE_SDK_FILES "${FIRMWARE_SDK_FOLDER}" "*.cpp")
RECURSIVE_FIND_FILE(FIRMWARE_SDK_C_FILES "${FIRMWARE_SDK_FOLDER}" "*.c")
RECURSIVE_FIND_FILE(MODEL_FILES "${MODEL_FOLDER}/tflite-model" "*.cpp")
RECURSIVE_FIND_FILE(PLATFORM_FILES "${EI_PLATFORM_FOLDER}/ingestion-sdk-platform/espressif_esp32" "*.cpp")
RECURSIVE_FIND_FILE(SENSORS_FILES "${EI_PLATFORM_FOLDER}/ingestion-sdk-platform/sensors" "*.cpp")
RECURSIVE_FIND_FILE(INGESTION_FILES "${EI_PLATFORM_FOLDER}/ingestion-sdk-c" "*.cpp")
RECURSIVE_FIND_FILE(INFERENCE_FILES "${EI_PLATFORM_FOLDER}/inference" "*.cpp")
RECURSIVE_FIND_FILE(MBEDTLS_FILES "${EI_PLATFORM_FOLDER}/mbedtls_hmac_sha256_sw/mbedtls/src" "*.c")

file(GLOB_RECURSE ASM_ESP_DSP
    "${EI_SDK_FOLDER}/porting/espressif/esp-dsp/*.S"
    "${EI_SDK_FOLDER}/porting/espressif/esp-dsp/*.s"
)

list(APPEND SOURCE_FILES ${C_FILES})
list(APPEND SOURCE_FILES ${CC_FILES})
list(APPEND SOURCE_FILES ${PORTING_FILES})
list(APPEND SOURCE_FILES ${FIRMWARE_SDK_FILES})
list(APPEND SOURCE_FILES ${FIRMWARE_SDK_C_FILES})
list(APPEND SOURCE_FILES ${MODEL_FILES})
list(APPEND SOURCE_FILES ${PLATFORM_FILES})
list(APPEND SOURCE_FILES ${SENSORS_FILES})
list(APPEND SOURCE_FILES ${INGESTION_FILES})
list(APPEND SOURCE_FILES ${INFERENCE_FILES})
list(APPEND SOURCE_FILES ${MBEDTLS_FILES})
list(APPEND SOURCE_FILES ${ASM_ESP_DSP})

idf_component_register(SRCS "main.cpp" "${SOURCE_FILES}"
                       INCLUDE_DIRS "${include_dirs}"
                       REQUIRES ESP-NN driver esp_driver_gpio esp_driver_uart esp_adc nvs_flash esp_timer esp32-camera LIS3DHTR_ESP-IDF)

target_compile_options(${COMPONENT_LIB} PRIVATE -std=gnu++17)