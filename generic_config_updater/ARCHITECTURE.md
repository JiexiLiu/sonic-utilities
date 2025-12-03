# Generic Config Updater 模块架构详解

## 目录
1. [模块概述](#模块概述)
2. [核心组件](#核心组件)
3. [类详解](#类详解)
4. [工作流程](#工作流程)
5. [设计模式](#设计模式)

---

## 模块概述

**Generic Config Updater (GCU)** 是 SONiC 中用于安全、可靠地更新配置的核心模块。它提供了一套完整的机制来：
- 应用 JSON Patch 格式的配置更新
- 验证配置的正确性（基于 YANG 模型）
- 管理配置依赖关系
- 提供配置回滚和检查点功能
- 确保配置更新的原子性和一致性

### 主要文件
```
generic_config_updater/
├── generic_updater.py      # 主入口和工厂类
├── gu_common.py            # 通用工具和包装类
├── patch_sorter.py         # 补丁排序和验证
├── change_applier.py       # 配置应用器
├── services_validator.py   # 服务验证器
├── field_operation_validators.py  # 字段操作验证
└── *.conf.json            # 配置文件
```

---

## 核心组件

### 1. **GenericUpdater** - 主入口类
```python
class GenericUpdater:
    """
    提供配置更新的统一接口
    """
    def apply_patch(patch, config_format, verbose, dry_run,
                    ignore_non_yang_tables, ignore_paths, sort=True)
    def replace(target_config, ...)
    def rollback(checkpoint_name, ...)
    def checkpoint(checkpoint_name, ...)
    def delete_checkpoint(checkpoint_name, ...)
    def list_checkpoints(includes_time, ...)
```

**职责**：
- 作为外部调用的统一入口
- 委托给 GenericUpdateFactory 创建具体的执行器
- 支持多种操作：补丁应用、配置替换、回滚、检查点管理

---

### 2. **GenericUpdateFactory** - 工厂类
```python
class GenericUpdateFactory:
    """
    创建各种配置更新组件的工厂
    """
    def create_patch_applier(...)
    def create_config_replacer(...)
    def create_config_rollbacker(...)
    def get_config_wrapper(dry_run)
    def get_change_applier(dry_run, config_wrapper)
    def get_patch_sorter(ignore_non_yang_tables, ignore_paths, ...)
```

**职责**：
- 根据参数创建合适的组件实例
- 处理 dry-run 模式
- 配置组件之间的依赖关系

**创建的组件**：
- `PatchApplier`: 应用 JSON Patch
- `ConfigReplacer`: 替换整个配置
- `FileSystemConfigRollbacker`: 回滚到检查点
- `ConfigWrapper` / `DryRunConfigWrapper`: 配置访问包装器
- `ChangeApplier` / `DryRunChangeApplier`: 变更应用器
- `StrictPatchSorter` / `NonStrictPatchSorter`: 补丁排序器

---

### 3. **PatchApplier** - 补丁应用器
```python
class PatchApplier:
    """
    应用 JSON Patch 到 ConfigDB
    """
    def __init__(patchsorter, changeapplier, config_wrapper,
                 patch_wrapper, scope)
    def apply(patch, sort=True)
```

**工作流程**：
1. 获取当前配置
2. 模拟应用补丁生成目标配置
3. 验证字段操作的合法性
4. 验证目标配置不包含空表
5. 排序补丁（如果 sort=True）
6. 逐个应用变更
7. 验证最终配置与目标配置一致

**关键组件**：
- `patchsorter`: 对补丁进行排序以确保正确的应用顺序
- `changeapplier`: 实际应用单个变更到 ConfigDB
- `config_wrapper`: 访问和验证配置
- `patch_wrapper`: 补丁操作工具

---

### 4. **ConfigWrapper** - 配置包装器
```python
class ConfigWrapper:
    """
    封装 ConfigDB 访问和 YANG 验证
    """
    def get_config_db_as_json()
    def convert_config_db_to_sonic_yang(config_db_as_json)
    def convert_sonic_yang_to_config_db(sonic_yang_as_json)
    def validate_sonic_yang_config(sonic_yang_as_json)
    def validate_config_db_config(config_db_as_json)
    def validate_field_operation(old_config, target_config)
    def crop_tables_without_yang(config_db_as_json)
    def create_sonic_yang_with_loaded_models()
```

**核心功能**：
1. **格式转换**：
   - ConfigDB JSON ? SonicYang JSON

2. **配置验证**：
   - YANG 模型验证
   - 字段操作验证（禁止某些字段的添加/删除/替换）
   - 补充验证（lanes、BGP peer group 等）

3. **YANG 管理**：
   - 延迟加载 YANG 模型（首次使用时加载）
   - 缓存已加载的 SonicYang 实例

**补充验证器**：
```python
def validate_lanes(config_db)
    # 验证 PORT lanes 的唯一性和有效性

def validate_bgp_peer_group(config_db)
    # 验证 BGP_PEER_RANGE 中 IP 范围不重复
```

---

### 5. **PatchWrapper** - 补丁包装器
```python
class PatchWrapper:
    """
    处理 JSON Patch 操作
    """
    def validate_config_db_patch_has_yang_models(patch)
    def generate_patch(current, target)
    def simulate_patch(patch, jsonconfig)
    def convert_config_db_patch_to_sonic_yang_patch(patch)
    def convert_sonic_yang_patch_to_config_db_patch(patch)
```

**功能**：
- 验证补丁涉及的表都有 YANG 模型
- 生成两个配置之间的补丁
- 模拟应用补丁
- ConfigDB Patch ? SonicYang Patch 转换

---

### 6. **PathAddressing** - 路径寻址
```python
class PathAddressing:
    """
    处理 JSON Pointer 和 XPath 之间的转换
    """
    @staticmethod
    def get_path_tokens(path)

    @staticmethod
    def create_path(tokens)

    @staticmethod
    def get_xpath_tokens(xpath)

    def convert_path_to_xpath(path, config, sy)
    def convert_xpath_to_path(xpath, config, sy)
    def find_ref_paths(paths, config, reload_config=True)
    def configdb_sorted_keys_by_backlinks(configdb_path, configdb,
                                          reverse=False, ...)
```

**核心功能**：
1. **路径操作**：
   - 分割/连接 JSON Pointer 路径
   - 分割/连接 XPath

2. **路径转换**：
   - JSON Pointer → XPath
   - XPath → JSON Pointer

3. **依赖查找**：
   - `find_ref_paths()`: 查找引用指定路径的所有路径
   - 示例：查找所有引用 `/PORT/Ethernet0` 的配置

4. **智能排序**：
   - `configdb_sorted_keys_by_backlinks()`: 按反向链接数量排序键
   - 用于确定配置更新的顺序

---

## 补丁排序系统

### 7. **PatchSorter** - 补丁排序器接口
```python
class PatchSorter(ABC):
    @abstractmethod
    def sort(patch) -> list[JsonChange]
```

### 8. **StrictPatchSorter** - 严格排序器
```python
class StrictPatchSorter(PatchSorter):
    """
    严格模式：确保所有配置都有 YANG 模型并进行完整验证
    """
    def __init__(config_wrapper, patch_wrapper)
    def sort(patch)
```

**排序策略**：
1. 将 JSON Patch 转换为 `Diff` 对象
2. 使用 `MoveWrapper` 生成所有可能的 `JsonMove`
3. 通过 BFS 搜索找到从当前配置到目标配置的最优路径
4. 每个移动都经过验证器验证
5. 返回排序后的 `JsonChange` 列表

**关键特性**：
- 保证最小破坏性更新
- 处理配置依赖关系
- 验证每一步的有效性

### 9. **NonStrictPatchSorter** - 非严格排序器
```python
class NonStrictPatchSorter(PatchSorter):
    """
    非严格模式：允许忽略某些表或路径
    """
    def __init__(config_wrapper, patch_wrapper, config_splitter)
    def sort(patch)
```

**工作流程**：
1. 使用 `ConfigSplitter` 分割配置：
   - 有 YANG 模型的配置
   - 无 YANG 模型的配置（如果 ignore_non_yang_tables=True）
   - 忽略路径的配置（如果 ignore_paths 指定）
2. 对有 YANG 模型的配置使用 `StrictPatchSorter`
3. 对其他配置直接转换为 `JsonChange`

---

## Diff 和 Move 系统

### 10. **Diff** - 配置差异
```python
class Diff:
    """
    表示当前配置和目标配置之间的差异
    """
    def __init__(current_config, target_config)
    def apply_move(move, in_place=False)
    def undo_move(move, in_place=False)
    def has_no_diff()
```

**用途**：
- 封装配置转换的起点和终点
- 支持应用和撤销移动操作
- 用于搜索算法中的状态表示

### 11. **JsonMove** - JSON 移动操作
```python
class JsonMove:
    """
    类似 JsonPatch，但允许路径引用不存在的中间元素
    """
    def __init__(diff, op_type, current_config_tokens,
                 target_config_tokens)
    def apply(config, in_place=False)
    def undo(config, in_place=False)

    @staticmethod
    def from_patch(patch)

    @staticmethod
    def from_operation(operation)
```

**与 JsonPatch 的区别**：
- JsonPatch: 路径必须完全存在
- JsonMove: 允许路径中的中间元素不存在

**示例**：
```json
// 当前配置
{}

// JsonPatch (会失败)
{"op": "add", "path": "/dict1/key11", "value": "value11"}
// 失败原因：dict1 不存在

// JsonMove (会成功)
// 自动转换为
{"op": "add", "path": "/dict1", "value": {"key11": "value11"}}
```

### 12. **JsonMoveGroup** - 移动组
```python
class JsonMoveGroup:
    """
    一组需要一起应用的 JsonMove
    """
    def __init__(move=None)
    def append(move)
    def apply(config, in_place=False)
    def undo(config, in_place=False)
    def get_jsonpatch()
    def parentTableName()
    def merge(group)
```

**用途**：
- 将相关的移动操作组合在一起
- 支持批量应用和撤销
- 用于优化更新性能

---

## Move 生成和验证系统

### 13. **MoveWrapper** - 移动包装器
```python
class MoveWrapper:
    """
    生成、扩展和验证移动操作
    """
    def __init__(move_generators, move_non_extendable_generators,
                 move_extenders, move_validators)
    def generate(diff)
    def validate(move, diff)
    def simulate(move, diff, in_place=False)
```

**组件**：
1. **move_generators**: 生成可扩展的移动
2. **move_non_extendable_generators**: 生成不可扩展的移动（高级操作）
3. **move_extenders**: 扩展移动（例如，尝试替换父配置）
4. **move_validators**: 验证移动的有效性

**生成流程**：
```
1. 生成不可扩展移动（如删除/添加整个表）
2. 生成可扩展移动（低级操作）
3. 扩展移动（尝试不同的变体）
4. 对每个移动进行验证
```

---

## 移动验证器

### 14. **FullConfigMoveValidator** - 完整配置验证器
```python
class FullConfigMoveValidator:
    """
    验证应用移动后的完整配置符合 YANG 模型
    """
    def validate(move, diff, simulated_config)
```

### 15. **CreateOnlyMoveValidator** - 只创建字段验证器
```python
class CreateOnlyMoveValidator:
    """
    验证只创建字段只能在创建时设置，不能修改
    """
    def validate(group, diff, simulated_config)
```

**只创建字段示例**：
- `PORT.lanes`: 端口通道只能在创建时设置
- `BGP_NEIGHBOR.asn`: BGP 邻居 ASN 不能修改
- `LOOPBACK_INTERFACE.vrf_name`: 环回接口 VRF 不能修改

**验证规则**：
- 字段不能被替换
- 字段只能在父对象创建时添加
- 字段只能在父对象删除时删除

### 16. **NoDependencyMoveValidator** - 无依赖验证器
```python
class NoDependencyMoveValidator:
    """
    验证单个移动内的配置没有相互依赖
    """
    def validate(group, diff, simulated_config)
```

**验证逻辑**：
- 对于 ADD 操作：检查添加的配置内部没有依赖
- 对于 REMOVE 操作：检查删除的配置内部没有依赖
- 对于 REPLACE 操作：检查添加和删除的配置之间没有依赖

**示例**：
```
无效的移动：
- 同时添加 PORT 和引用该 PORT 的 VLAN_MEMBER
- 同时删除 PORT 和引用该 PORT 的 VLAN_MEMBER

有效的移动：
- 先删除 VLAN_MEMBER，再删除 PORT
- 先添加 PORT，再添加 VLAN_MEMBER
```

### 17. **NoEmptyTableMoveValidator** - 非空表验证器
```python
class NoEmptyTableMoveValidator:
    """
    验证移动不会导致空表（ConfigDB 中空表不显示）
    """
    def validate(group, diff, simulated_config)
```

### 18. **RequiredValueMoveValidator** - 必需值验证器
```python
class RequiredValueMoveValidator:
    """
    验证某些配置需要其他字段为特定值
    """
    def validate(group, diff, simulated_config)
```

**示例规则**：
```python
{
    "required_pattern": ["PORT", "@", "admin_status"],
    "required_value": "down",
    "requiring_patterns": [
        ["QUEUE", "@|*"],
        ["BUFFER_PG", "@|*"],
        ...
    ]
}
```

**含义**：
- 修改 `QUEUE` 或 `BUFFER_PG` 时，对应的 `PORT.admin_status` 必须为 `down`
- 如果还有 `QUEUE` 变更待处理，不能将 `PORT.admin_status` 改为 `up`

### 19. **DeleteWholeConfigMoveValidator** - 删除整个配置验证器
```python
class DeleteWholeConfigMoveValidator:
    """
    禁止删除整个配置（JsonPatch 库不支持）
    """
    def validate(group, diff, simulated_config)
```

### 20. **RemoveCreateOnlyDependencyMoveValidator** - 删除只创建依赖验证器
```python
class RemoveCreateOnlyDependencyMoveValidator:
    """
    验证修改只创建字段前，所有依赖已被删除
    """
    def validate(group, diff, simulated_config)
```

---

## 辅助类

### 21. **JsonPointerFilter** - JSON 指针过滤器
```python
class JsonPointerFilter:
    """
    使用模式匹配从配置中获取路径
    """
    def __init__(patterns, path_addressing)
    def get_paths(config, common_key=None)
    def is_match(path)
```

**模式语法**：
- `*`: 匹配任意键
- `@`: 替换为 common_key
- `*|@`: 匹配以 `|common_key` 结尾的键
- `@|*`: 匹配以 `common_key|` 开头的键

**示例**：
```python
patterns = [["PORT", "*", "admin_status"]]
# 匹配：/PORT/Ethernet0/admin_status, /PORT/Ethernet4/admin_status, ...

patterns = [["VLAN_MEMBER", "@|*"]]
common_key = "Vlan100"
# 匹配：/VLAN_MEMBER/Vlan100|Ethernet0, /VLAN_MEMBER/Vlan100|Ethernet4, ...
```

### 22. **RequiredValueIdentifier** - 必需值识别器
```python
class RequiredValueIdentifier:
    """
    识别需要其他字段为特定值的配置
    """
    def __init__(path_addressing)
    def get_required_value_data(configs)
    def get_value_or_default(config, path)
    def target_in_required_pattern(configdb_path_tokens)
```

**数据结构**：
```python
{
    "/QUEUE/Ethernet0|0": [
        ("/PORT/Ethernet0/admin_status", "down"),
        ...
    ],
    ...
}
```

### 23. **CreateOnlyFilter** - 只创建过滤器
```python
class CreateOnlyFilter:
    """
    过滤只创建字段
    """
    def __init__(path_addressing)
    def get_filter()
```

**硬编码的只创建字段**：
```python
[
    ["PORT", "*", "lanes"],
    ["LOOPBACK_INTERFACE", "*", "vrf_name"],
    ["BGP_NEIGHBOR", "*", "holdtime"],
    ["BGP_NEIGHBOR", "*", "keepalive"],
    ["BGP_NEIGHBOR", "*", "name"],
    ["BGP_NEIGHBOR", "*", "asn"],
    ...
]
```

---

## 变更应用系统

### 24. **ChangeApplier** - 变更应用器
```python
class ChangeApplier:
    """
    将 JsonChange 应用到 ConfigDB
    """
    def __init__(scope)
    def apply(current_configdb, change)
    def remove_backend_tables_from_config(data)
```

**工作流程**：
1. 应用 JsonChange 到当前配置（内存中）
2. 计算需要更新的表和键
3. 调用服务验证器
4. 使用 `ConfigDBConnector.set_entry()` 更新 Redis
5. 等待 1 秒（避免竞态条件）
6. 返回更新后的配置

**后端表**：
```python
backend_tables = [
    "BUFFER_PG",
    "BUFFER_PROFILE",
    "FLEX_COUNTER_TABLE"
]
```
这些表由后端服务自动管理，不应在验证时比较。

### 25. **DryRunChangeApplier** - 模拟变更应用器
```python
class DryRunChangeApplier:
    """
    模拟应用变更（不实际写入 ConfigDB）
    """
    def __init__(config_wrapper)
    def apply(current_configdb, change)
```

---

## 服务验证系统

### 26. **服务验证器函数**

#### rsyslog_validator
```python
def rsyslog_validator(old_config, upd_config, keys):
    """
    验证 SYSLOG_SERVER 变更并重启 rsyslog 服务
    """
```

#### dhcp_validator
```python
def dhcp_validator(old_config, upd_config, keys):
    """
    重启 dhcp_relay 服务
    """
```

#### vlan_validator
```python
def vlan_validator(old_config, upd_config, keys):
    """
    如果 VLAN 的 dhcp_servers 变更，重启 dhcp_relay
    """
```

#### caclmgrd_validator
```python
def caclmgrd_validator(old_config, upd_config, keys):
    """
    如果控制平面 ACL 规则变更，等待 1 秒让 caclmgrd 更新 iptables
    """
```

#### ntp_validator
```python
def ntp_validator(old_config, upd_config, keys):
    """
    重启 chrony 服务
    """
```

#### vlanintf_validator
```python
def vlanintf_validator(old_config, upd_config, keys):
    """
    如果 VLAN 接口 IP 被删除，刷新邻居表
    """
```

**配置文件** (`gcu_services_validator.conf.json`):
```json
{
    "tables": {
        "SYSLOG_SERVER": {
            "services_to_validate": ["rsyslog"]
        },
        "DHCP_RELAY": {
            "services_to_validate": ["dhcp_relay"]
        },
        ...
    },
    "services": {
        "rsyslog": {
            "validate_commands": [
                "generic_config_updater.services_validator.rsyslog_validator"
            ]
        },
        ...
    }
}
```

---

## 配置回滚系统

### 27. **FileSystemConfigRollbacker** - 文件系统配置回滚器
```python
class FileSystemConfigRollbacker:
    """
    管理配置检查点和回滚
    """
    def __init__(checkpoints_dir, config_replacer, config_wrapper, scope)
    def rollback(checkpoint_name)
    def checkpoint(checkpoint_name)
    def list_checkpoints(includes_time=False)
    def delete_checkpoint(checkpoint_name)
```

**检查点存储**：
- 目录：`/etc/sonic/checkpoints/`
- 格式：`{checkpoint_name}.cp.json`
- 内容：完整的 ConfigDB JSON

**操作**：
1. **checkpoint**: 保存当前配置到文件
2. **rollback**: 从文件加载配置并替换当前配置
3. **list_checkpoints**: 列出所有检查点（可选包含时间戳）
4. **delete_checkpoint**: 删除指定检查点

### 28. **MultiASICConfigRollbacker** - 多 ASIC 配置回滚器
```python
class MultiASICConfigRollbacker(FileSystemConfigRollbacker):
    """
    支持多 ASIC 系统的配置回滚
    """
    def rollback(checkpoint_name)
    def checkpoint(checkpoint_name)
```

**特性**：
- 为每个命名空间（host + asic0, asic1, ...）分别保存/恢复配置
- 检查点文件包含所有命名空间的配置

---

## 装饰器模式

### 29. **Decorator** - 基础装饰器
```python
class Decorator(PatchApplier, ConfigReplacer, FileSystemConfigRollbacker):
    """
    装饰器基类，用于扩展功能
    """
    def __init__(decorated_patch_applier, decorated_config_replacer,
                 decorated_config_rollbacker, scope)
```

### 30. **SonicYangDecorator** - SonicYang 装饰器
```python
class SonicYangDecorator(Decorator):
    """
    支持 SonicYang 格式的配置
    """
    def apply(patch)  # 将 SonicYang patch 转换为 ConfigDB patch
    def replace(target_config)  # 将 SonicYang 配置转换为 ConfigDB
```

### 31. **ConfigLockDecorator** - 配置锁装饰器
```python
class ConfigLockDecorator(Decorator):
    """
    为配置操作添加锁保护
    """
    def apply(patch, sort=True)
    def replace(target_config)
    def rollback(checkpoint_name)
    def checkpoint(checkpoint_name)
```

**注意**：当前 `ConfigLock` 是空实现（TODO）

---

## 工作流程详解

### 应用补丁流程

```
1. GenericUpdater.apply_patch()
   ↓
2. GenericUpdateFactory.create_patch_applier()
   ↓
3. PatchApplier.apply()
   ├─ 3.1 获取当前配置
   ├─ 3.2 模拟应用补丁
   ├─ 3.3 验证字段操作
   ├─ 3.4 验证无空表
   ├─ 3.5 排序补丁 (StrictPatchSorter/NonStrictPatchSorter)
   │      ├─ 创建 Diff
   │      ├─ MoveWrapper.generate() 生成所有可能的移动
   │      ├─ BFS 搜索最优路径
   │      └─ 每个移动通过验证器验证
   ├─ 3.6 逐个应用变更 (ChangeApplier)
   │      ├─ 计算需要更新的键
   │      ├─ 更新 ConfigDB
   │      ├─ 调用服务验证器
   │      └─ 等待 1 秒
   └─ 3.7 验证最终配置
```

### 配置替换流程

```
1. GenericUpdater.replace()
   ↓
2. GenericUpdateFactory.create_config_replacer()
   ↓
3. ConfigReplacer.replace()
   ├─ 3.1 获取当前配置
   ├─ 3.2 生成补丁 (current → target)
   ├─ 3.3 调用 PatchApplier.apply()
   └─ 3.4 验证最终配置
```

### 配置回滚流程

```
1. GenericUpdater.rollback()
   ↓
2. GenericUpdateFactory.create_config_rollbacker()
   ↓
3. FileSystemConfigRollbacker.rollback()
   ├─ 3.1 验证检查点存在
   ├─ 3.2 加载检查点配置
   ├─ 3.3 调用 ConfigReplacer.replace()
   └─ 3.4 验证最终配置
```

---

## 设计模式

### 1. **工厂模式** (Factory Pattern)
- `GenericUpdateFactory` 负责创建各种组件
- 根据参数（dry_run, ignore_non_yang_tables 等）创建不同的实现

### 2. **策略模式** (Strategy Pattern)
- `PatchSorter` 接口定义排序策略
- `StrictPatchSorter` 和 `NonStrictPatchSorter` 提供不同的实现

### 3. **装饰器模式** (Decorator Pattern)
- `SonicYangDecorator` 添加 SonicYang 格式支持
- `ConfigLockDecorator` 添加锁保护

### 4. **模板方法模式** (Template Method Pattern)
- `PatchApplier.apply()` 定义了应用补丁的模板流程
- 具体步骤委托给其他组件

### 5. **组合模式** (Composite Pattern)
- `JsonMoveGroup` 将多个 `JsonMove` 组合在一起

### 6. **状态模式** (State Pattern)
- `Diff` 表示配置转换的状态
- `JsonMove` 表示状态转换

### 7. **责任链模式** (Chain of Responsibility)
- 多个验证器按顺序验证移动
- 任何一个验证器失败，移动就被拒绝

---

## 关键算法

### BFS 搜索算法（补丁排序）

```python
def _sort(diff):
    queue = [(diff, [])]  # (当前diff, 已应用的变更列表)
    visited = {diff}

    while queue:
        current_diff, changes = queue.pop(0)

        if current_diff.has_no_diff():
            return changes  # 找到解决方案

        for move in move_wrapper.generate(current_diff):
            if not move_wrapper.validate(move, current_diff):
                continue

            new_diff = move_wrapper.simulate(move, current_diff)
            if new_diff in visited:
                continue

            visited.add(new_diff)
            new_changes = changes + [JsonChange(move.get_jsonpatch())]
            queue.append((new_diff, new_changes))

    raise Exception("无法找到有效的更新路径")
```

**特点**：
- 保证找到最短路径（最少的变更步骤）
- 每个状态都经过完整验证
- 避免重复访问相同状态

---

## 配置示例

### 应用简单补丁

```python
from generic_config_updater.generic_updater import GenericUpdater
from generic_config_updater.generic_updater import ConfigFormat
import jsonpatch

# 创建补丁
patch = jsonpatch.JsonPatch([
    {"op": "add", "path": "/PORT/Ethernet0", "value": {"alias": "eth0"}}
])

# 应用补丁
updater = GenericUpdater()
updater.apply_patch(
    patch=patch,
    config_format=ConfigFormat.CONFIGDB,
    verbose=True,
    dry_run=False,
    ignore_non_yang_tables=False,
    ignore_paths=None,
    sort=True
)
```

### 创建和回滚检查点

```python
# 创建检查点
updater.checkpoint("before_change", verbose=True)

# 应用变更
updater.apply_patch(...)

# 回滚
updater.rollback("before_change", verbose=True, dry_run=False,
                 ignore_non_yang_tables=False, ignore_paths=None)
```

### 替换配置

```python
# 加载目标配置
with open("target_config.json") as f:
    target_config = json.load(f)

# 替换
updater.replace(
    target_config=target_config,
    config_format=ConfigFormat.CONFIGDB,
    verbose=True,
    dry_run=False,
    ignore_non_yang_tables=False,
    ignore_paths=None
)
```

---

## 总结

Generic Config Updater 是一个复杂但设计精良的系统，它提供了：

1. **安全性**：通过 YANG 验证和多层验证器确保配置正确性
2. **可靠性**：通过补丁排序和依赖管理避免配置冲突
3. **灵活性**：支持多种配置格式和操作模式
4. **可维护性**：清晰的模块划分和设计模式
5. **可扩展性**：易于添加新的验证器和服务验证

**核心优势**：
- 自动处理配置依赖关系
- 最小化配置更新的破坏性
- 提供完整的回滚机制
- 支持 dry-run 模式进行测试

**适用场景**：
- 配置管理系统
- 自动化部署工具
- 配置迁移工具
- 配置验证工具
