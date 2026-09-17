# FoodTraceability · 食品溯源区块链系统

基于 **FISCO BCOS 区块链平台 + Solidity 智能合约 + Spring Boot** 实现的食品溯源案例系统（GZ036 区块链技术应用赛项 · 第 1 套附件案例代码）。

系统以"**生产商 → 中间商 → 超市 → 消费者**"为业务主线，将食品在生产、分销、出售各环节的关键信息（时间、操作人、操作地址、食品质量）上链存证，实现从农场到餐桌的全程可追溯。

---

## 功能特点

- **链上溯源存证**：食品每一环节的流转信息（时间戳、操作人、操作地址、质量）均写入区块链，不可篡改
- **角色权限控制**：合约层内置 `Producer` / `Distributor` / `Retailer` 三种角色，只有被授权的角色才能执行对应操作（`onlyProducer` / `onlyDistributor` / `onlyRetailer` 修饰符校验）
- **流程状态机**：食品状态按 `生产(0) → 分销(1) → 出售(2)` 单向流转，合约内部强制校验，防止跳过环节
- **全链路查询**：支持按溯源 ID 查询完整溯源轨迹、查看某食品当前信息、按环节（生产/分销/出售中）筛选食品
- **四角色 Web 端**：农场（生产商）、中间商、超市、消费者四种角色的登录与操作页面

---

## 系统架构

```
┌─────────────────────────────────────────────────────────────┐
│                      前端页面（浏览器）                        │
│   农场 / 中间商 / 超市 / 消费者 四角色界面（Vue + Element UI）    │
└──────────────────────────┬──────────────────────────────────┘
                           │ HTTP
┌──────────────────────────▼──────────────────────────────────┐
│               Spring Boot 后端（本仓库 backend）               │
│   接口层：/produce /adddistribution /addretail /trace ...      │
│   合约调用层：构造交易参数（合约名/地址/ABI/函数/参数）并转发        │
└──────────────────────────┬──────────────────────────────────┘
                           │ POST /trans/handle
┌──────────────────────────▼──────────────────────────────────┐
│                     WeBASE-Front（前置服务）                   │
│              交易签名、发送上链、查询链上数据                    │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│               FISCO BCOS 区块链底层（链上合约）                │
│        Trace 主合约 + Producer/Distributor/Retailer 角色合约   │
│        + FoodInfoItem 食品信息合约（本仓库 contract）           │
└─────────────────────────────────────────────────────────────┘
```

---

## 技术栈

| 层级 | 技术 |
|------|------|
| 区块链底层 | FISCO BCOS（通过 WeBASE-Front 前置服务交互） |
| 智能合约 | Solidity `0.4.25 ~ 0.6.x`（ABIEncoderV2） |
| 后端 | Spring Boot 2.6.2、Java 1.8、Thymeleaf、FastJSON、Apache HttpClient |
| 前端 | Vue 2、Element UI、Axios、jQuery（静态资源方式引入） |

---

## 项目结构

```
FoodTraceability/
├── backend/                          # Spring Boot 后端
│   ├── conf.properties               # 运行配置（WeBASE-Front 地址、合约及角色地址）
│   ├── pom.xml                       # Maven 依赖配置
│   ├── mvnw / mvnw.cmd               # Maven Wrapper
│   └── src/main/
│       ├── java/com/zgxt/demo/
│       │   ├── DemoApplication.java      # 启动类
│       │   ├── config/MvcConfig.java     # 静态资源映射
│       │   └── controller/IndexController.java  # 核心接口（合约调用逻辑）
│       └── resources/
│           ├── application.properties     # HTTP 连接池配置
│           ├── templates/index.html       # 前端入口页（Thymeleaf）
│           └── static/                    # 前端静态资源（Vue/Element 库、四角色页面）
└── contract/                         # Solidity 智能合约
    ├── Roles.sol                     # 角色库（地址授权管理）
    ├── Producer.sol                  # 生产商角色合约
    ├── Distributor.sol               # 中间商角色合约
    ├── Retailer.sol                  # 超市角色合约
    ├── FoodInfoItem.sol              # 食品溯源信息合约
    └── Trace.sol                     # 溯源主合约（业务入口）
```

---

## 智能合约设计

| 合约 | 职责 |
|------|------|
| `Roles.sol` | 通用角色库，提供角色的 `add` / `remove` / `has` 地址授权管理 |
| `Producer.sol` | 生产商角色：构造函数注入初始生产商地址，`onlyProducer` 权限修饰符，支持添加/退出角色 |
| `Distributor.sol` | 中间商角色：同上的分销角色管理 |
| `Retailer.sol` | 超市角色：同上的零售角色管理 |
| `FoodInfoItem.sol` | 单个食品的溯源载体：保存各环节时间戳、用户名、用户地址、质量数组及食品基本信息，内部校验流转状态 |
| `Trace.sol` | 溯源主合约：继承三个角色合约，管理 `溯源ID → 食品合约` 的映射，提供生产/分销/出售/查询全部业务接口 |

### 核心接口（Trace 合约）

| 接口 | 权限 | 说明 |
|------|------|------|
| `newFood(name, traceNumber, traceName, quality)` | 仅生产商 | 新建食品溯源信息，返回食品合约地址 |
| `addTraceInfoByDistributor(traceNumber, traceName, quality)` | 仅中间商 | 分销环节追加溯源信息 |
| `addTraceInfoByRetailer(traceNumber, traceName, quality)` | 仅超市 | 出售环节追加溯源信息 |
| `getTraceInfo(traceNumber)` | 公开 | 获取完整溯源轨迹（时间/用户名/地址/质量 四组数组） |
| `getFood(traceNumber)` | 公开 | 获取食品基本信息及当前状态 |
| `getAllFood()` | 公开 | 获取全部食品溯源 ID 列表 |

### 状态与质量约定

- **流转状态**：`0` = 生产中，`1` = 分销中，`2` = 已出售（必须按序流转）
- **食品质量**：`0` = 优质，`1` = 合格，`2` = 不合格

---

## 后端接口说明

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/index` | 系统首页（四角色入口） |
| GET | `/userinfo?userName=producer\|distributor\|retailer` | 获取指定角色的链上地址 |
| POST | `/produce` | 生产商录入食品信息（body：`traceNumber`、`foodName`、`traceName`、`quality`） |
| POST | `/adddistribution` | 中间商追加分销溯源信息（body：`traceNumber`、`traceName`、`quality`） |
| POST | `/addretail` | 超市追加出售溯源信息（body：`traceNumber`、`traceName`、`quality`） |
| GET | `/foodlist` | 获取所有食品基本信息 |
| GET | `/food?traceNumber=` | 获取某食品当前信息 |
| GET | `/trace?traceNumber=` | 获取某食品完整溯源轨迹 |
| GET | `/newtracelist` | 获取所有食品的最新溯源信息 |
| GET | `/producing` | 处于生产环节的食品列表 |
| GET | `/distributing` | 处于分销环节的食品列表 |
| GET | `/retailing` | 处于出售环节的食品列表 |

---

## 快速开始

### 1. 环境准备

| 依赖 | 说明 |
|------|------|
| JDK 1.8+ | 后端运行环境 |
| Maven 3.x | 后端构建（也可直接使用仓库内 `mvnw`） |
| FISCO BCOS 区块链节点 | 底层链环境 |
| WeBASE-Front | 区块链前置服务（默认端口 5002，提供交易上链与查询接口） |

### 2. 部署智能合约

1. 启动 FISCO BCOS 链与 WeBASE-Front，确保 `http://<节点IP>:5002/WeBASE-Front` 可访问
2. 使用 Remix 或 WeBASE 控制台编译 `contract/` 下全部合约
3. 部署 `Trace.sol`，构造函数依次传入三个参数：
   - `producer`：生产商地址
   - `distributor`：中间商地址
   - `retailer`：超市地址
4. 记录部署后的合约地址

### 3. 修改后端配置

编辑 `backend/conf.properties`（需放在后端运行目录，即 `mvn spring-boot:run` 的执行目录）：

```properties
# WeBASE-Front 交易处理地址
URL=http://<节点IP>:5002/WeBASE-Front/trans/handle
# Trace 合约地址（部署后获得）
CONTRACT_ADDRESS=0x...
# 生产商 / 中间商 / 超市 用户地址
PRODUCER_ADDRESS=0x...
DISTRIBUTOR_ADDRESS=0x...
RETAILER_ADDRESS=0x...
```

> 仓库内默认配置为竞赛演示环境的示例值，实际运行请替换为你的链上地址。

### 4. 启动后端

```bash
cd backend
mvn spring-boot:run
```

### 5. 访问系统

浏览器打开 `http://localhost:8080/index`，分别以 **农场（生产商）→ 中间商 → 超市** 身份登录体验完整溯源流程，消费者角色可查询食品溯源信息。

---

## 使用流程演示

```
1. 农场登录，录入新食品（生产）        → 链上记录：时间、生产商、质量
2. 中间商登录，对该食品追加分销信息    → 链上追加：时间、中间商、质量（状态 生产→分销）
3. 超市登录，对该食品追加出售信息      → 链上追加：时间、超市、质量（状态 分销→出售）
4. 消费者输入溯源ID查询               → 展示完整溯源轨迹（各环节时间/操作人/质量）
```

---

## 说明

- 本仓库为 **GZ036 区块链技术应用赛项 · 第 1 套附件案例代码**，仅用于教学与竞赛训练
- 合约地址、角色地址、节点 IP 等均为演示环境配置，正式使用请按实际环境修改
- 后端通过 WeBASE-Front 完成交易签名与上链，需保证后端与前置服务网络互通
