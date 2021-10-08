
## cartographer 3d 相对与 2d 的不同点

### 三维网格地图
ActiveSubmaps3D
Submap3D
HybridGrid

### 前端

#### LocalTrajectoryBuilder3D

#### RealTimeCorrelativeScanMatcher3D

#### CeresScanMatcher3D

### 后端

#### PoseGraph3D

基本一样, 

第一个子图只优化yaw角
是否优化z坐标

直接获取local_pose, 不用进行旋转变换了

#### ConstraintBuilder3D
基本一样,

调用的3d的 FastCorrelativeScanMatcher3D 与 CeresScanMatcher3D

#### FastCorrelativeScanMatcher3D











## 调参指导

### 降低延迟与减小计算量

  前端部分
    减小 max_range, 减小了需要处理的点数, 在雷达数据远距离的点不准时一定要减小这个值
    增大 voxel_filter_size, 相当于减小了需要处理的点数
    增大 submaps.resolution, 相当于减小了匹配时的搜索量
    对于自适应体素滤波 减小 min_num_points与max_range, 增大 max_length, 相当于减小了需要处理的点数
  
  后端部分
    减小 optimize_every_n_nodes, 降低优化频率, 减小了计算量
    增大 MAP_BUILDER.num_background_threads, 增加计算速度
    减小 global_sampling_ratio, 减小计算全局约束的频率
    减小 constraint_builder.sampling_ratio, 减少了约束的数量
    增大 constraint_builder.min_score, 减少了约束的数量
    减小分枝定界搜索窗的大小, 包括linear_xy_search_window,inear_z_search_window, angular_search_window
    增大 global_constraint_search_after_n_seconds, 减小计算全局约束的频率
    减小 max_num_iterations, 减小迭代次数

### 降低内存
  增大子图的分辨率 submaps.resolution

### 常调的参数
  TRAJECTORY_BUILDER_2D.min_range = 0.3
  TRAJECTORY_BUILDER_2D.max_range = 100.
  TRAJECTORY_BUILDER_2D.min_z = 0.2 / -0.8
  TRAJECTORY_BUILDER_2D.voxel_filter_size = 0.02

  TRAJECTORY_BUILDER_2D.ceres_scan_matcher.occupied_space_weight = 10.
  TRAJECTORY_BUILDER_2D.ceres_scan_matcher.translation_weight = 1.
  TRAJECTORY_BUILDER_2D.ceres_scan_matcher.rotation_weight = 1.

  TRAJECTORY_BUILDER_2D.submaps.num_range_data = 80.
  TRAJECTORY_BUILDER_2D.submaps.grid_options_2d.resolution = 0.1 / 0.02

  POSE_GRAPH.optimize_every_n_nodes = 160. 2倍的num_range_data以上
  POSE_GRAPH.constraint_builder.sampling_ratio = 0.3
  POSE_GRAPH.constraint_builder.max_constraint_distance = 15.
  POSE_GRAPH.constraint_builder.min_score = 0.48
  POSE_GRAPH.constraint_builder.global_localization_min_score = 0.60

## 工程化建议

### 工程化的目的

根据机器人的**传感器硬件**, 最终能够实现**稳定地**构建一张**不叠图**的二维栅格地图.

由于cartographer代码十分的优秀, 所以cartographer的稳定性目前已经很好了, 比目前的大部分slam的代码都稳定, 很少会出现崩掉的情况, 最多就是会由于某些原因提示错误.

### 如何提升建图质量

最简单的一种方式, 选择好的传感器. 选择频率高(25hz以上), 精度高的雷达, 精度高的imu, 这样的传感器配置下很难有建不好的地图.

##### 如果只能用频率低的雷达呢?

由于频率低时的叠图基本都是在旋转时产生的, 所以推荐使用一个好的imu, 然后建图的时候让机器人的**移动与旋转速度慢一点**(建图轨迹与建图速度十分影响建图效果), 这时候再看建图效果.

如果效果还不行, 调ceres的匹配权重, 将地图权重调大, 平移旋转权重调小. 
如果效果还不行, 可以将代码中平移和旋转的残差注释掉.
如果效果还不行, 那就得改代码了, 去改位姿推测器那部分的代码, 让预测的准一点.

##### 里程计

为什么一直没有说里程计, 就是由于cartographer中对里程计的使用不太好.

cartographer中对里程计的使用有2部分, 一个是前端的位姿推测器, 一个是后端根据里程计数据计算残差. 后端部分的使用是没有问题的.

如果想要在cartographer中使用里程计达到比较好的效果, 前端的位姿推测器这部分需要自己重写. 

可以将karto与gmapping的使用里程计进行预测的部分拿过来进行使用, 改完了之后就能够达到比较好的位姿预测效果了.

##### 粗匹配
cartographer的扫描匹配中的粗匹配是一种暴力匹配的方法, 目的是对位姿预测出的位姿进行校准, 但是这个扫描匹配的计算量太大了, 导致不太好用.

这块可以进行改进, 可以将karto的扫描匹配的粗匹配放过来, karto的扫描匹配的计算量很小, 当做粗匹配很不错.

##### 地图

有时前端部分生成的地图出现了叠图, 而前端建的地图在后端是不会被修改的, 后端优化只会优化节点位姿与子图位姿.

同时cartographer_ros最终生成的地图是将所有地图叠加起来的, 就会导致这个叠图始终都存在, 又或者是后边的地图的空白部分将前边的地图的边给覆盖住了, 导致墙的黑边消失了.

后端优化会将节点与子图的位姿进行优化, 但是不会改动地图, 所以可以在最终生成地图的时候使用后端优化后的节点重新生成一次地图, 这样生成的地图的效果会比前端地图的叠加要好很多.

这块的实现可以参考一下我写的实时生成三维点云地图部分的代码.

更极致的修改

后端优化后的节点与子图位姿是不会对前端产生影响的, 这块可以进行优化一下, 就是前端匹配的时候, 不再使用前端生成的地图进行匹配, 而是使用后端生成的地图进行匹配, 这样就可以将后端优化后的效果带给前端. 但是这要对代码进行大改, 比较费劲.

### 降低计算量与内存
- 体素滤波与自适应体素滤波的计算量(不是很大)
- 后端进行子图间约束时的计算量很大
- 分支定界算法的计算量很大

- 降低内存, 内存的占用基本就是多分辨率地图这, 每个子图的多分辨率地图都进行保存是否有必要


### 纯定位的改进建议
目前cartographer的纯定位和正常的建图是一样的, 只是仅保存3个子图, 依然要进行后端优化.

这就导致了几个问题:

第一个: 前端的扫描匹配, 是当前的雷达点云与当前轨迹的地图进行匹配, 而不是和之前的地图进行匹配, 这就导致了定位时机器人当前的点云与之前的地图不一定能匹配很好, 就是因为当前的点云是匹配当前轨迹的地图的, 不是与之前的地图进行匹配.

第二个: 纯定位其实就是建图, 所以依然会进行回环检测与后端优化, 而后端优化的计算在定位这是没有必要的, 带来了额外的计算量.

第三个: 纯定位依然会进行回环检测, 回环检测有可能导致机器人的位姿发生跳变, 


#### 纯定位
#### 重定位

### 去ros的参考思路
有一些公司不用ros, 所以就要进行去ros的开发.

咱讲过数据是怎么通过cartographer_ros传到cartographer里去的, 只要仿照着cartographer_ros里的操作, 获取到传感器数据, 将数据转到tracking_frame坐标系下并进行格式转换, 再传入到cartographer里就行了.

cartographer_ros里使用ros的地方比较少, 只有在node.cc, sensor_bridge等几个类中进行使用, 只需要改这个类接受数据的方式以及将ros相关的格式修改一下就行了.


---

第一章 编译运行及调参指导
1. Cartorgapher论文带读
2. 代码的编译与运行
3. 配置文件参数详解与调试
4. 代码基础介绍

a. cartographer中使用的c++11新标准的语法
b. 关键概念与关键数据结构
第二章 cartographer_ros代码阅读
1. 入口函数分析

a. gflags简介
b. glog简介
c. 自定义log的格式
d. 配置文件的加载
2. ROS接口

a. SLAM的启动, 停止, 数据的保存与加载
b. 话题的订阅与发布
c. 传感器数据的走向与处理
d. 服务的处理
e. 可视化信息的设置与发布
f. ROS中的数据类型与cartorgrapher中的数据类型的转换
3. ROS地图的保存与发布

a. 子地图与ROS地图间的格式转换
b. ROS地图的发布
第三章 传感器数据的处理过程分析
1. 传感器数据的传递过程分析

a. Cartographer_ros中的传感器数据的传递过程分析
b. Cartographer中的传感器数据的传递过程分析
2. 2D情况下的激光雷达数据的预处理

a. 多个激光雷达数据的时间同步与融合
b. 激光雷达数据运动畸变的校正与无效点的处理
c. 将点云据根据重力的方向做旋转变换
d. 对变换后的点云进行z方向的过滤
e. 对过滤后的点云做体素滤波
3. 3D情况下的激光雷达数据的预处理

a. 多个激光雷达数据的时间同步与融合
b. 对融合后的点云进行第一次体素滤波
c. 激光雷达数据运动畸变的校正与无效点的处理
d. 分别对点云的hit点与miss点进行第二次体素滤波
e. 将点云根据预测出来的先验位姿做旋转平移变换
第四章 2D扫描匹配
1. 扫描匹配理论过程分析
2. 生成2D局部地图

a. 查找表的实现
b. 概率地图的实现与地图坐标系的定义
c. 概率地图的更新方式
d. 如何将点云插入到概率地图中
3. 基于局部地图的扫描匹配

a. 基于IMU与里程计的先验位姿估计
b. 根据机器人当前姿态将预测出来的6维先验位姿投影成水平面下的3维位姿
c. 对之前预处理后的点云进行自适应体素滤波
d. 使用实时的相关性扫描匹配进行粗匹配
e. ceres的简介与编程练习
f. 进行基于图优化的扫描匹配实现精匹配
4. 扫描匹配的结果的处理

a. 将匹配的结果与生成的局部地图加入到后端的位姿图结构中
b. 调用传入的回调函数进行扫描匹配结果与局部地图的保存
第五章 2D 后端优化
1. 任务队列与线程池
2.

向位姿图中添加顶点
为顶点进行子图内与子图间约束的计算
3. 2D情况下的回环检测

a. 使用滑动窗口算法生成指定分辨率的栅格地图
b. 多分辨率栅格地图的生成
c. 对点云做不同角度的旋转得到各个角度下的点云
d. 生成最低分辨率地图下的所有的可能解
e. 对所有的可能解进行评分与排序
f. 使用分支定界算法找到最优解
4. 2D情况下的优化问题的构建
第六章 代码总结与3D建图
1. 代码的全面总结
2.

TSDF
4.4 生成3D局部地图

a. 查找表的实现
b. 3D网格地图的实现
c. 3D网格地图的更新方式
d. 如何将点云插入到3D网格地图中
4.5 3D情况下的基于局部地图的扫描匹配

a. 基于IMU与里程计的先验位姿估计
b. 将点云分别进行高分辨率与低分辨率的体素滤波
c. 使用实时的相关性扫描匹配对高分辨率体素滤波后的点云进行粗匹配
d. 进行基于图优化的扫描匹配实现精匹配
e. 旋转直方图的作用
f. 基于ceres-solver的优化模型搭建
5.3 3D情况下的后端优化

a. 基于ceres-solver的后端位姿图优化模型的搭建
b. 向位姿图中添加基于GPS的2个连续位姿间的约束
c. 向位姿图中添加基于IMU的角速度和线性加速度的约束
d. 残差项雅克比矩阵的计算与优化模型的求解
6.3 3D情况下的回环检测

a. 多分辨率网格地图的生成
b. 计算点云在低分辨率地图下的得分作为初值
c. 通过旋转直方图对点云所有可能的旋转角度进行评分
d. 根据评分是否大于阈值生成按照可能角度旋转后的点云
e. 生成最低分辨率地图下的所有的可能解
f. 对所有的可能解进行评分与排序
g. 使用分支定界算法找到最优解
第七章 调参总结
1. 降低延迟与减小计算量
2. 降低内存
3. 常调的参数
第八章 工程化建议
1. Cartographer优缺点总结
2. 如何提升建图质量
3. 降低计算量与内存
4. 纯定位的改进建议
5. 去ros的参考思路