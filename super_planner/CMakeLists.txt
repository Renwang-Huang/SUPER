cmake_minimum_required(VERSION 3.5)
project(super_planner)

# 解决 PCL_ROOT 相关的 CMake 警告
if(POLICY CMP0074)
  cmake_policy(SET CMP0074 NEW)
endif()

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 1. 编译选项：开启 Release 模式，并静默警告
set(CMAKE_BUILD_TYPE Release)
# set(CMAKE_CXX_FLAGS "-O3 -Wall -g -fPIC -Wno-sign-compare -Wno-parentheses -Wno-reorder -Wno-maybe-uninitialized -Wno-unused-variable -Wno-unused-but-set-variable -Wno-deprecated-declarations")
add_compile_options(-Werror=return-type)

set(CMAKE_CXX_FLAGS "-O3 -w -g -fPIC")

# 2. 宏定义
add_definitions(-DROOT_DIR=\"${CMAKE_CURRENT_SOURCE_DIR}/\")
add_definitions(-DORIGIN_AT_CORNER)
add_definitions(-DFMT_HEADER_ONLY)
add_definitions(-DUSE_ROS2)

string(TOUPPER $ENV{ROS_DISTRO} ROS_VERSION)
message(STATUS "ROS version: ${ROS_VERSION}")

# 3. 寻找构建工具和 ROS 2 基础依赖
find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(std_msgs REQUIRED)
find_package(geometry_msgs REQUIRED)
find_package(sensor_msgs REQUIRED)
find_package(nav_msgs REQUIRED)
find_package(visualization_msgs REQUIRED)
find_package(tf2_ros REQUIRED)
find_package(pcl_conversions REQUIRED)

# 寻找自定义 ROS 2 包
find_package(rog_map REQUIRED)
find_package(mars_quadrotor_msgs REQUIRED)

# 寻找第三方依赖
find_package(PCL REQUIRED)
find_package(Eigen3 REQUIRED)
find_package(yaml-cpp REQUIRED)

# 4. 依赖项分类整理
set(ros_libs
        rclcpp
        std_msgs
        geometry_msgs
        sensor_msgs
        nav_msgs
        visualization_msgs
        tf2_ros
        pcl_conversions
        rog_map
        mars_quadrotor_msgs
)

set(third_party_libs
        ${PCL_LIBRARIES}
        yaml-cpp
        -ldw
)

set(includes
        include
        ${Eigen3_INCLUDE_DIRS}
        ${PCL_INCLUDE_DIRS}
)

include_directories(${includes})


#================= FOR libsuper ============================#
file(GLOB_RECURSE srcs CONFIGURE_DEPENDS src/*.cpp include/*.h include/*.hpp)

# 生成静态库
add_library(super STATIC ${srcs})

target_include_directories(super PUBLIC ${includes})
# 【关键修复】：删除了 PUBLIC 关键字，保持和 ament_target_dependencies 一致的 Plain 风格
target_link_libraries(super ${third_party_libs})
ament_target_dependencies(super ${ros_libs})


#================= FOR Benchmark Node ======================#
add_executable(fsm_node Apps/fsm_node_ros2.cpp)

# 【关键修复】：删除了 PUBLIC 关键字
target_link_libraries(fsm_node 
        super
        ${third_party_libs}
)
ament_target_dependencies(fsm_node ${ros_libs})


#================= INSTALL all targets =====================#

# 1. 安装可执行文件到 lib/包名 下 (ROS 2 标准做法)
install(TARGETS fsm_node
        DESTINATION lib/${PROJECT_NAME}
)

# 2. 安装头文件目录
install(DIRECTORY include/
        DESTINATION include/
)

# 3. 安装 launch 和 config 文件夹到 share 下
install(DIRECTORY launch config
        DESTINATION share/${PROJECT_NAME}/
)

# 4. 导出依赖项 (供其它包依赖当前包时使用)
ament_export_include_directories(include)
ament_export_dependencies(${ros_libs})

ament_package()