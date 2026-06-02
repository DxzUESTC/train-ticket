# Bug Fix Record - Consign Service (托运服务)

## Overview

修复了 `ts-consign-service` 和 `ts-consign-price-service` 中的 6 个 bug，涵盖 Repository 返回类型错误、HTTP 调用无异常处理、空指针隐患、代码质量问题。

---

## Bug 1: `findByOrderId` 返回类型错误导致多记录时抛异常

**严重程度:** 严重  
**文件:** `ts-consign-service/src/main/java/consign/repository/ConsignRepository.java`  
**问题:** `findByOrderId` 声明返回单个 `ConsignRecord`，但 `orderId` 在表中无唯一约束，多条记录时 Spring Data JPA 抛出 `IncorrectResultSizeDataAccessException`。  

**修复:**
- `ConsignRecord findByOrderId(String accountId)` → `ArrayList<ConsignRecord> findByOrderId(String orderId)`
- `ConsignServiceImpl.queryByOrderId()` 改为 list 判断（`!= null && !isEmpty()`），与 `queryByConsignee`/`queryByAccountId` 对齐
- `ConsignServiceImplTest` 新增空 list 测试用例

**影响文件:** `ConsignRepository.java`, `ConsignServiceImpl.java`, `ConsignServiceImplTest.java`

---

## Bug 2: 调用价格服务无异常处理，高并发下频繁失败

**严重程度:** 严重  
**文件:** `ts-consign-service/src/main/java/consign/service/ConsignServiceImpl.java`  
**问题:** `restTemplate.exchange()` 调用价格服务无 try-catch、无空值检查、无超时配置，任何网络/服务异常直接抛 500。

**修复:**
- `insertConsignRecord()` 和 `updateConsignRecord()` 中的价格服务调用添加 try-catch，异常时返回 `Response(0, ...)` 而非抛异常
- `re.getBody()` 和 `getData()` 添加空值判断
- `ConsignApplication` 中 RestTemplate 添加 `connectTimeout=5s` / `readTimeout=10s`
- `ConsignPriceServiceImpl.getPriceByWeightAndRegion()` 添加 `priceConfig == null` 防御，避免级联 NPE

**影响文件:** `ConsignServiceImpl.java`, `ConsignApplication.java`, `ConsignPriceServiceImpl.java`

---

## Bug 3: 对可能为 null 的 String 字段调用 `.toString()`

**严重程度:** 中等  
**文件:** `ts-consign-service/src/main/java/consign/service/ConsignServiceImpl.java`  
**问题:** `consignRequest.getOrderId().toString()` 和 `consignRequest.getAccountId().toString()` — 字段已是 String，多余的 `.toString()` 在值为 null 时导致 NPE。无 `@Valid`/@NotNull 校验。

**修复:** 3 处 `.toString()` 调用直接移除，直接用字段值赋值。

---

## Bug 4: `double` 类型用 `!=` 直接比较

**严重程度:** 低  
**文件:** `ts-consign-service/src/main/java/consign/service/ConsignServiceImpl.java`  
**问题:** `originalRecord.getWeight() != consignRequest.getWeight()` — IEEE 754 浮点精度问题可能导致误判，触发不必要的价格重算。

**修复:** 改为 `Double.compare(originalRecord.getWeight(), consignRequest.getWeight()) != 0`

---

## Bug 5: `Consign` DTO 类含废弃 JPA 注解

**严重程度:** 轻微  
**文件:** `ts-consign-service/src/main/java/consign/entity/Consign.java`  
**问题:** 类上有 `@GenericGenerator`，字段上有 `@Id`/`@GeneratedValue`/`@Column`，但无 `@Entity` 注解，所有 JPA 注解均为死代码。

**修复:** 移除所有 JPA 注解及相关 import (`javax.persistence.*`, `org.hibernate.annotations.GenericGenerator`)

---

## Bug 6: pom.xml name 标签错误

**严重程度:** 轻微  
**文件:** `ts-consign-service/pom.xml`  
**问题:** `<name>ts-consign-price-service</name>` 应为 `<name>ts-consign-service</name>`

**修复:** 修正 name 值。

---

## Summary

| Bug | Severity | Files Changed |
|-----|----------|--------------|
| Bug 1: `findByOrderId` return type | Critical | 3 |
| Bug 2: No error handling on price call | Critical | 3 |
| Bug 3: `.toString()` on nullable String | Medium | 1 |
| Bug 4: `!=` on double comparison | Low | 1 |
| Bug 5: Orphaned JPA annotations | Trivial | 1 |
| Bug 6: pom.xml name typo | Trivial | 1 |
