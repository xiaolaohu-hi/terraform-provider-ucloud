# MySQL 产品接入设计（Terraform Provider）

- 日期：2026-09-21
- 状态：待评审
- 类型：新产品接入（按《Terraform Provider 产品接入与跨产品协作规范》第 6 章）

## 1. 背景与目标

为 Terraform Provider 新增 **MySQL** 产品包，管理 UCloud **nvme 新机型**的 MySQL 实例。与现有 `udb` 产品（`ucloud_db_*`，SATA 老机型）并行，互不冲突。

目标范围（已与产品方确认）：

1. 1 个资源 `ucloud_mysql_instance`：Create / Read / Delete（**无 Update**，所有可变字段 `ForceNew`；stop 并入 delete 流程）。
2. 3 个只读数据源：`ucloud_mysql_instances`、`ucloud_mysql_parameter_templates`、`ucloud_mysql_machine_types`。
3. **仅 nvme 新机型**：硬编码 `StorageClass=CLOUD_RSSD`、`SpecificationClass=O`（后端映射为 `InstanceType=NVMe_SSD`）。
4. SDK 调用方式：**generic invoke**（nvme 接口未进 typed SDK，与 ucloud-cli 一致）。

## 2. 命名

| 项 | 值 |
|---|---|
| 产品包 | `products/mysql` |
| 产品名（Key） | `mysql`（主数据 EnSampleName `MySQL`） |
| Terraform 命名空间 | `ucloud_mysql_*`（`terraformNamespaces=["mysql"]`） |
| 资源 | `ucloud_mysql_instance` |
| 数据源 | `ucloud_mysql_instances` / `ucloud_mysql_parameter_templates` / `ucloud_mysql_machine_types` |

## 3. 架构与调用链

```
terraform provider（generic invoke，public_key/private_key + region/project_id）
   │  Action ∈ {CreateUDBMySQLInstance, DescribeUDBInstance,
   │            StopUDBInstance, DeleteUDBInstance,
   │            ListUDBParamTemplate, ListUDBMachineType}
   ▼
UCloud OpenAPI 总入口 udb-api-gateway（Node.js，按 Action 分发）
   ├─ create/stop/delete → micro/api（nvme 管控，HTTP→RPC）→ udb.srv.mysql.core（gRPC）
   └─ describe/state      → legacy udb_access RPC（同份 DB 记录，含 udb_generation 富化）
```

- 鉴权复用 provider 现有 `public_key/private_key + region/project_id`；`CompanyID(top_organization_id)/AccountID(organization_id)/zone_id` 是网关内部翻译，provider 侧不感知。
- 网关侧 create 的 `CreateUDBMySQLInstance` 会改写为 micro/api 的 `CreateUDBInstance`；describe 走 legacy RPC，不走 micro/api（micro 的公开 `DescribeUDBInstance` 目前仅支持 MongoDB）。

### 动作清单（provider 通过 generic invoke 调用）

| 用途 | Action | 关键入参 | 关键返回 |
|---|---|---|---|
| create | `CreateUDBMySQLInstance` | Region/Zone/ProjectId, Name, AdminPassword, DBTypeId, Port, DiskSpace, ParamGroupId, MachineType, StorageClass=CLOUD_RSSD, SpecificationClass=O, ChargeType, Quantity, InstanceMode, VPCId, SubnetId, Tag | `DBId` |
| read/list | `DescribeUDBInstance` | Region/Zone/ProjectId, ClassType=sql, DBId（单查）/Offset/Limit（列表） | `DataSet[]`（DBId, Name, State, VirtualIP, Port, DBTypeId, MemoryLimit, DiskSpace, VPCId, SubnetId, InstanceMode, InstanceType, CreateTime, ExpiredTime…） |
| stop | `StopUDBInstance` | DBId | RetCode |
| delete | `DeleteUDBInstance` | DBId | RetCode |
| 参数模板 | `ListUDBParamTemplate` | DBVersion, Zone | `DataSet[]`（Id, Name…） |
| 机型列表 | `ListUDBMachineType` | Region/Zone, InstanceMode | `DataSet[]`（ID, Cpu, Memory, Description…） |

## 4. 文件清单

```
products/mysql/
├── product.go            # Name="mysql"；Registration 注册 1 资源 + 3 数据源
├── client.go             # generic invoke 客户端（ucloud.Client + NewGenericRequest/GenericInvoke）
├── resource_mysql_instance.go
├── data_source_mysql_instances.go
├── data_source_mysql_parameter_templates.go
├── data_source_mysql_machine_types.go
├── service.go            # describe/等待/错误转换
├── compat.go             # 状态常量、converter、校验器
└── *_test.go             # 注册/字段/CRUD/客户端/导入/验收测试

internal/productcatalog/catalog.go       # 登记 mysql（newAdapter=mysql.New, terraformNamespaces=["mysql"],
                                          #   masterData{EnSampleName:"MySQL", Key:"mysql"}, 测试基线）
.github/product-owners.json              # 所有权（产品团队用 product-ownership-sync 生成）
ucloud/testdata/provider_contract.json   # 接口基线（core 审查后重新生成）
examples/mysql/ + website/docs/{r,d}/mysql*  # 示例 + 用户文档
```

## 5. 资源 `ucloud_mysql_instance` 设计

### 5.1 Schema

| 字段 | 类型 | 属性 | 映射（API） | 说明 |
|---|---|---|---|---|
| `availability_zone` | string | Required, ForceNew | Zone | |
| `name` | string | Optional, Computed | Name | 未填自动生成 `tf-mysql-instance-` 前缀 |
| `db_version` | string | Optional, ForceNew | DBTypeId | 完整值，默认 `mysql-8.0` |
| `machine_type` | string | Required, ForceNew | MachineType | `o.mysql*`（用 ListUDBMachineType 校验） |
| `password` | string | Optional, Sensitive, Computed | AdminPassword | 未填自动生成 |
| `instance_mode` | string | Optional, ForceNew | InstanceMode | `Normal`/`HA`，默认 `HA` |
| `disk_space` | int | Required | DiskSpace(GB) | |
| `port` | int | Optional, Computed | Port | 默认 3306 |
| `charge_type` | string | Optional, ForceNew | ChargeType | `Month`/`Year`/`Dynamic`，默认 `Month` |
| `duration` | int | Optional, ForceNew | Quantity | 默认 1 |
| `param_group_id` | string | Optional, Computed | ParamGroupId | 未填用 ListUDBParamTemplate 取默认 |
| `vpc_id` | string | Optional, ForceNew | VPCId | |
| `subnet_id` | string | Optional, ForceNew | SubnetId | |
| `tag` | string | Optional, ForceNew | Tag | |
| `private_ip` | string | Computed | VirtualIP | |
| `status` | string | Computed | State | |
| `cpu` | int | Computed | CPU / CpuCoreCount | |
| `memory` | int | Computed | MemoryLimit(MB) / 1000 | 内存，单位 GB |
| `create_time` | string | Computed | CreateTime | RFC3339 |
| `expire_time` | string | Computed | ExpiredTime | RFC3339 |
| `modify_time` | string | Computed | ModifyTime | RFC3339 |

- `StorageClass=CLOUD_RSSD`、`SpecificationClass=O` 写死在 Create 里，不进 schema。
- `Timeouts`：Create 30m / Delete 10m（复用 udb 的经验值）。

### 5.2 CRUD

- **Create**：组 `CreateUDBMySQLInstance` 请求 → 取 `DBId` → `StateChangeConf` 轮询 `DescribeUDBInstance`，`Init/Starting → Running`，`Fail` 报错。密码未填则本地生成并 `d.Set("password", ...)`。
- **Read**：`DescribeUDBInstance` 按 `DBId` 单查 → 回填所有字段；空 `DataSet` 或 notfound → `d.SetId("")`。
- **Delete**：`State=Running` 时先 `StopUDBInstance` → 等 `Shutoff` → `DeleteUDBInstance` → 复查确认已删（delete 不会自动 stop）。
- **Update**：v1 无可变字段，所有字段 `ForceNew`，因此不声明 `Update` 回调（SDK 校验要求如此）。

### 5.3 状态机常量

`Init`/`Starting`=创建中、`Running`=就绪、`Shutoff`=已停、`Deleting`/`DeleteFail`/`Fail`/`RecoverFail`/`UpgradeFail` 等（来源 micro/api `MapDBStateToStr`）。

## 6. 数据源设计

| 数据源 | Action | 入参 | 返回列表元素字段 |
|---|---|---|---|
| `ucloud_mysql_instances` | `DescribeUDBInstance` | availability_zone, ids, name_regex, vpc_id, output_file | id, name, zone, status, private_ip, port, db_version, memory, disk_space, instance_mode, charge_type, create_time, expire_time |
| `ucloud_mysql_parameter_templates` | `ListUDBParamTemplate` | db_version, zone, output_file | id, name, db_version, is_default |
| `ucloud_mysql_machine_types` | `ListUDBMachineType` | instance_mode, zone, output_file | id, cpu, memory, description |

- 分页：`limit=100` + `offset` 递增，直到返回 <100（沿用现有 data source 模式）。
- `d.SetId(hashStringArray(ids))` + `total_count`（沿用现有模式）。

## 7. client.go（generic invoke）设计

```go
// clientFromMeta 断言 product.RuntimeV1，经 runtime.ProductClient(Name, newClient)
// 返回 *ucloud.Client（SDK 基础客户端，非某 service 的 typed client）。
//
// newClient 满足接入规范约束：无外部副作用、复制 Config 副本、挂载所有 handler、非 nil。
func newClient(config *ucloud.Config, credential *auth.Credential,
    handlers []ucloud.HttpRequestHandler) interface{} {
    cfg := *config
    cfg.Timeout = 60 * time.Second
    c := ucloud.NewClient(&cfg, credential)
    for _, h := range handlers { c.AddHttpRequestHandler(h) }
    return c
}

// invoke 封装 generic invoke：SetPayload(map) + GenericInvoke + 反序列化。
func invoke(client *ucloud.Client, action string, payload map[string]interface{}) (map[string]interface{}, error)
```

- `Region`/`ProjectId` 取自 client 配置，`Zone` 取资源 `availability_zone`，`Action` + 业务字段放 payload。

## 8. 错误处理

- notfound：`DescribeUDBInstance` 空 DataSet → `newNotFoundError`（沿用 `isNotFoundError` 约定）。
- API 错误：`RetCode != 0` → 透传 `Message` 为 error，含资源 ID 便于人工恢复。
- 重试：Delete 用 `resource.Retry` 包 stop→delete 序列（沿用 udb 模式）。

## 9. 测试与接入配套（按接入规范）

1. 注册测试、字段定义单元测试、CRUD 单元测试、客户端测试（RuntimeV1/构造器/类型断言）。
2. 导入测试 `import_ucloud_mysql_instance_test.go`（`ImportStatePassthrough`）。
3. 验收测试 `TestAccUCloudMySQLInstance_*`（真实云环境，nvme 机型，产品团队承担费用）。
4. `make compat`、`make test`、`make vet` 通过。
5. catalog 条目 + `.github/product-owners.json`（产品团队用 `product-ownership-sync` 生成）+ `provider_contract.json` 接口基线（core 审查）。
6. `examples/mysql/` 最小示例 + `website/docs/r/mysql_instance.html.markdown` + `website/docs/d/mysql_*.html.markdown`。

## 10. 风险与备注

- **describe 走 legacy RPC 而非 micro/api**：字段以 gateway `DescribeUDBInstance.js` 返回的 DataSet 为准；`SSDType`/`InstanceType` 返回 `NVMe`/`NVMe_SSD`。
- **参数模板**：create 用 `ListUDBParamTemplate`（与 conf_list 的 `DescribeUDBParamGroup` 不同，后者为旧接口，不用）。
- **机型校验**：`machine_type` 合法性依赖 `ListUDBMachineType` 返回，Create 前可校验；校验失败报清晰错误。
- 后端 `DescribeUDBInstance` 公开动作对非 MongoDB 暂不支持，故 provider **必须走 gateway 的 `DescribeUDBInstance`**（legacy RPC 路径），不能直连 micro/api 的公开 describe。
