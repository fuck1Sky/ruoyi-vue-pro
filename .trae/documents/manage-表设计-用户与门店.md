# 管理（manage）表设计方案：用户与门店

## Summary
- 在现有项目表设计规范（统一审计字段 + 逻辑删除）基础上，新增 2 张以 `manage_` 为前缀的表：
  - 用户表（`manage_user`）
  - 门店表（`manage_store`）
- 按用户要求，不添加 `tenant_id`，且核心字段包含 `id`、`name` 即可。

## Proposed Changes
### 1) 新增表：manage_user（用户表）
**目的**
- 记录系统用户基本信息。

**字段设计（MySQL）**
- 主键：`id` bigint auto_increment (主键编号)
- 业务标识：`user_id` bigint (用户编号)
- 核心信息：`name` varchar(100) - 用户名称
- 审计及通用字段：`creator`, `create_time`, `updater`, `update_time`, `deleted`

### 2) 新增表：manage_store（门店表）
**目的**
- 记录系统门店基本信息。

**字段设计（MySQL）**
- 主键：`id` bigint auto_increment (主键编号)
- 业务标识：`store_id` bigint (门店编号)
- 核心信息：`name` varchar(100) - 门店名称
- 审计及通用字段：`creator`, `create_time`, `updater`, `update_time`, `deleted`

## MySQL DDL

```sql
-- ----------------------------
-- Table structure for manage_user
-- ----------------------------
DROP TABLE IF EXISTS `manage_user`;
CREATE TABLE `manage_user`  (
  `id` bigint NOT NULL AUTO_INCREMENT COMMENT '主键编号',
  `user_id` bigint NOT NULL DEFAULT 0 COMMENT '用户编号',
  `name` varchar(100) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL DEFAULT '' COMMENT '用户名称',
  `creator` varchar(64) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NULL DEFAULT '' COMMENT '创建者',
  `create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `updater` varchar(64) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NULL DEFAULT '' COMMENT '更新者',
  `update_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted` bit(1) NOT NULL DEFAULT b'0' COMMENT '是否删除',
  PRIMARY KEY (`id`) USING BTREE,
  INDEX `idx_user_id`(`user_id` ASC) USING BTREE,
  INDEX `idx_create_time`(`create_time` ASC) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 1 CHARACTER SET = utf8mb4 COLLATE = utf8mb4_unicode_ci COMMENT = '用户表';

-- ----------------------------
-- Records of manage_user
-- ----------------------------
BEGIN;
COMMIT;

-- ----------------------------
-- Table structure for manage_store
-- ----------------------------
DROP TABLE IF EXISTS `manage_store`;
CREATE TABLE `manage_store`  (
  `id` bigint NOT NULL AUTO_INCREMENT COMMENT '主键编号',
  `store_id` bigint NOT NULL DEFAULT 0 COMMENT '门店编号',
  `name` varchar(100) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL DEFAULT '' COMMENT '门店名称',
  `creator` varchar(64) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NULL DEFAULT '' COMMENT '创建者',
  `create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `updater` varchar(64) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NULL DEFAULT '' COMMENT '更新者',
  `update_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted` bit(1) NOT NULL DEFAULT b'0' COMMENT '是否删除',
  PRIMARY KEY (`id`) USING BTREE,
  INDEX `idx_store_id`(`store_id` ASC) USING BTREE,
  INDEX `idx_create_time`(`create_time` ASC) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 1 CHARACTER SET = utf8mb4 COLLATE = utf8mb4_unicode_ci COMMENT = '门店表';

-- ----------------------------
-- Records of manage_store
-- ----------------------------
BEGIN;
COMMIT;
```
