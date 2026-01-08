# 磁带管理系统架构图

```mermaid
graph TD
    %% 工具层
    subgraph "工具层"
        log["log()"]
    end

    %% 资源层
    subgraph "资源层"
        MediaType["MediaType\n- LTO8\n- LTO9"]
        TapeStatus["TapeStatus\n- IN_LIBRARY\n- MOUNTED\n- EXPORTED"]
        Tape["Tape\n- barcode\n- media_type\n- status\n- metadata"]
        Slot["Slot\n- index\n- tape"]
        Drive["Drive\n- serial\n- media_type\n- mounted\n- load()\n- unload()"]
        TapeLibrary["TapeLibrary\n- slots\n- drives\n- find_empty_slot()\n- find_barcode()\n- move_to_drive()\n- move_to_slot()"]
    end

    %% 能力层
    subgraph "能力层"
        TapeRepository["TapeRepository\n- find_by_barcode()"]
        DriveManager["DriveManager\n- lease_drive()\n- release()"]
        Robotics["Robotics\n- move_to_drive()\n- move_to_slot()"]
    end

    %% Job/Task 模型
    subgraph "Job/Task 模型"
        JobType["JobType\n- DUPLICATE\n- VERIFY\n- FORMAT"]
        TaskType["TaskType\n- MOVE_TAPE\n- LOAD_TAPE\n- UNLOAD_TAPE\n- DUP_READ_WRITE"]
        TaskStatus["TaskStatus\n- PENDING\n- RUNNING\n- SUCCESS\n- FAILED"]
        Task["Task\n- id\n- type\n- status\n- payload\n- retry_left"]
        Job["Job\n- id\n- type\n- status\n- tasks"]
    end

    %% 调度器
    subgraph "调度器"
        TaskHandler["TaskHandler\n- handle()"]
        JobScheduler["JobScheduler\n- submit()\n- run()"]
    end

    %% 业务层
    subgraph "业务层"
        DuplicationService["DuplicationService\n- submit_dup_job()"]
    end

    %% 启动层
    subgraph "启动层"
        main["main()"]
    end

    %% 组件关系
    main --> TapeLibrary
    main --> TapeRepository
    main --> DriveManager
    main --> Robotics
    main --> TaskHandler
    main --> JobScheduler
    main --> DuplicationService
    
    TapeRepository --> TapeLibrary
    DriveManager --> TapeLibrary
    Robotics --> TapeLibrary
    
    TaskHandler --> TapeRepository
    TaskHandler --> DriveManager
    TaskHandler --> Robotics
    
    JobScheduler --> TaskHandler
    
    DuplicationService --> JobScheduler
    DuplicationService --> DriveManager
    
    JobScheduler --> Job
    Job --> Task
    
    Drive --> Tape
    Drive --> TapeStatus
    Slot --> Tape
    TapeLibrary --> Slot
    TapeLibrary --> Drive
    
    Tape --> MediaType
    Tape --> TapeStatus
    
    log -.-> "所有组件" 
```

## 架构说明

### 1. 工具层
- 提供基础日志功能，供所有组件使用

### 2. 资源层
- **MediaType**：定义磁带类型枚举
- **TapeStatus**：定义磁带状态枚举
- **Tape**：磁带实体类，包含条码、类型、状态和元数据
- **Slot**：槽位实体类，包含索引和当前磁带
- **Drive**：驱动器实体类，包含序列号、支持的媒体类型、当前挂载的磁带，以及加载/卸载方法
- **TapeLibrary**：磁带库核心类，管理槽位和驱动器，提供磁带查找、移动等基础操作

### 3. 能力层
- **TapeRepository**：磁带仓库，封装磁带查找功能
- **DriveManager**：驱动器管理器，负责驱动器的租赁和释放
- **Robotics**：机械手模拟，封装磁带在槽位和驱动器之间的移动操作

### 4. Job/Task 模型
- **JobType**：作业类型枚举（复制、验证、格式化）
- **TaskType**：任务类型枚举（移动磁带、加载磁带、卸载磁带、复制读写）
- **TaskStatus**：任务状态枚举（待处理、运行中、成功、失败）
- **Task**：任务实体类，包含任务ID、类型、状态、负载、重试次数
- **Job**：作业实体类，包含作业ID、类型、状态和任务列表

### 5. 调度器
- **TaskHandler**：任务处理器，将具体任务映射到设备动作
- **JobScheduler**：作业调度器，负责任务的调度和执行，处理任务状态更新和重试逻辑

### 6. 业务层
- **DuplicationService**：复制服务，提供提交复制作业的接口

### 7. 启动层
- **main()**：程序入口，负责初始化所有组件并启动系统

## 数据流
1. 业务层通过服务接口提交作业
2. 调度器接收作业，分解为任务并管理任务状态
3. 任务处理器将任务映射到具体的设备操作
4. 能力层封装底层资源的访问
5. 资源层直接操作磁带库的硬件资源

## 设计特点
1. **分层架构**：清晰的层次划分，各层职责明确
2. **模块化设计**：组件之间低耦合，便于扩展和维护
3. **任务驱动**：基于作业和任务的模型，支持复杂工作流
4. **状态管理**：完善的状态机设计，便于监控和调试
5. **线程安全**：使用锁机制保证并发安全
6. **可扩展性**：支持多种磁带类型、作业类型和任务类型

这个架构图清晰展示了磁带管理系统的各个组件及其关系，便于理解系统的整体设计和工作流程。
