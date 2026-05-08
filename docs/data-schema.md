# 数据文件格式规范 (Version 2)

本文档描述 `seats-tool.html` 导出/导入 JSON 文件的数据结构。符合此规范的 JSON 文件可直接导入工具，还原完整的画布设计。

## 快速示例

```json
{
  "version": 2,
  "groups": [
    { "id": "g_tech", "name": "技术部", "color": "#3498db" },
    { "id": "g_hr", "name": "人事部", "color": "#e74c3c" }
  ],
  "people": [
    { "id": "p_lichunping", "name": "李春平", "groupId": "g_tech" }
  ],
  "seats": [
    { "id": "s_xxx", "x": 120, "y": 90, "orientation": 0, "personId": "p_lichunping" }
  ],
  "obstacles": [
    { "id": "o_yyy", "x": 300, "y": 150, "w": 60, "h": 60, "name": "柱子 A" }
  ],
  "activeGroupId": "g_tech"
}
```

## 顶层字段

| 字段 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `version` | `number` | 是 | Schema 版本号，当前为 `2` |
| `groups` | `Array<Group>` | 是 | 所有人员分组 |
| `people` | `Array<Person>` | 是 | 所有人员（无论是否入座） |
| `seats` | `Array<Seat>` | 是 | 所有已放置的座位 |
| `obstacles` | `Array<Obstacle>` | 是 | 所有已放置的障碍物 |
| `activeGroupId` | `string` | 是 | 当前处于激活状态的分组 ID |

> 导入时如果 version 为 1，工具会自动进行数据迁移：根据人员原有颜色创建分组，并将人员关联至对应分组。

---

## 实体定义

### Group（分组）

| 字段 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `id` | `string` | 是 | 唯一标识 |
| `name` | `string` | 是 | 分组名称 |
| `color` | `string` | 是 | 分组统一颜色，Hex 格式 |

### Person（人员）

| 字段 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `id` | `string` | 是 | 唯一标识 |
| `name` | `string` | 是 | 显示名称 |
| `groupId` | `string` | 是 | 所属分组的 ID |

> v2 版本中人员不再拥有独立颜色，颜色由所属分组决定。

### Seat（座位）

| 字段 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `id` | `string` | 是 | 唯一标识 |
| `x` | `number` | 是 | 左上角 X 坐标 |
| `y` | `number` | 是 | 左上角 Y 坐标 |
| `orientation` | `number` | 是 | 朝向角度 (0, 90, 180, 270) |
| `personId` | `string|null` | 是 | 占用此座位的人员 ID |

### Obstacle（障碍物）

| 字段 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `id` | `string` | 是 | 唯一标识 |
| `x` | `number` | 是 | 左上角 X 坐标 |
| `y` | `number` | 是 | 左上角 Y 坐标 |
| `w` | `number` | 是 | 宽度 |
| `h` | `number` | 是 | 高度 |
| `name` | `string` | 是 | 障碍物名称 |

---

## 迁移逻辑 (v1 -> v2)

当导入 v1 格式数据时：
1. 遍历 `people` 数组。
2. 对于每种唯一的 `color`，创建一个对应的 `Group` 实体。
3. 将该颜色的所有 `Person` 实体的 `groupId` 设置为新创建的分组 ID。
4. 删除人员实体中的 `color` 字段。
5. 移除顶层的 `nextColorIdx` 字段。
6. 设置 `activeGroupId` 为第一个创建的分组 ID。
