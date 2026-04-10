# P3: Entry Schema 验证（Tape Entry Schema Validation）

> 灵感来源：TapeAgents Pydantic `Step` 类型体系 + `kind` 判别字段

## 问题

`TapeEntry.Payload` 是 `map[string]any`，写入时无任何验证。开发和调试中：
- 插件往 payload 塞了错误结构的数据，只在消费端（`ReadMessages`）才发现
- 不同版本的 entry 格式不兼容，但没有显式 schema 检查
- 工具结果格式不一致（有的返回 string，有的返回 map），消费端需要大量类型断言

## 方案

保留 `map[string]any` 存储层不变。新增可选的 schema 注册表：

```go
// pkg/tape/schema.go

type EntrySchemaRegistry struct {
    schemas map[string]map[string]any  // kind → JSON Schema
}

func (r *EntrySchemaRegistry) Register(kind string, schema map[string]any) {
    r.schemas[kind] = schema
}

func (r *EntrySchemaRegistry) Validate(entry TapeEntry) error {
    schema, ok := r.schemas[entry.Kind]
    if !ok { return nil }  // 未注册 schema 的 kind 不验证
    return jsonschema.Validate(schema, entry.Payload)
}
```

- 插件在 `Init()` 中注册自己 entry kind 的 schema
- 开发模式（`DMR_DEBUG=1`）下 `Append` 自动验证
- 生产模式下跳过验证（零开销）
- 新增 `DecodeAs[T](entry)` 泛型辅助函数

## 解决的问题

- 插件开发时更快发现 payload 格式错误
- tape 数据导出/分析时有 schema 参考
- 跨版本兼容性检查

## 参考

- TapeAgents `Step.kind` 判别体系：`tapeagents/core.py:72-97`
