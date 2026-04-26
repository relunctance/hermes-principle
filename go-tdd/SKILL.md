---
名称: go-tdd
描述: >
  Go 语言测试驱动开发规范。适用任何 Go 项目开发。
  触发时机：写任何 Go 代码之前自动加载。
版本: 1.0.0
作者: maomao
标签: [Go, TDD, 测试, 开发]
---

# Go TDD 开发规范

## 核心原则

```
RED  → 写失败的测试
GREEN → 写最简代码通过测试
REFACTOR → 清理
```

**铁律：没有失败的测试，就不能写生产代码。**

## TDD 循环

### 1. RED — 写失败的测试

```bash
# 先写测试文件
# 验证编译失败或测试失败
go test ./pkg/xxx/... -v -run TestXxx
# 预期：测试失败（功能未实现）
```

### 2. GREEN — 最少代码

写通过测试的最简代码。不要添加测试未覆盖的功能。

```bash
# 验证通过
go test ./pkg/xxx/... -v -run TestXxx
# 预期：测试通过
```

### 3. REFACTOR — 清理

只在 green 后消除重复、改进命名。

## hawk-memory 项目专用

详见 `hawk-memory-dev` skill。简要流程：

```bash
# 1. 写集成测试（HTTP 调用，无法 mock CGO）
# 2. go test → 失败
# 3. 实现功能
# 4. go build ./cmd/hawk-memory
# 5. systemctl --user restart hawk-memory
# 6. curl 端到端验证
# 7. git commit
```

**禁止用 kill 停止服务**：`systemctl --user restart hawk-memory`

## 测试类型

| 类型 | 适用场景 | 工具 |
|------|---------|------|
| 单元测试 | 接口可 mock | `go test ./...` |
| 集成测试 | HTTP API、DB、CGO | `go test ./... -tags=integration` |
| 基准测试 | 性能关键路径 | `go test ./... -bench=. -benchmem` |

## 测试命令速查

```bash
# 单个测试
go test ./pkg/xxx/... -v -run TestName

# 所有测试
go test ./... -v -count=1

# 带覆盖率
go test ./... -cover -count=1

# 编译检查（不运行）
go build ./...

# 竞争检测
go test ./... -race
```

## 常见陷阱

| 陷阱 | 解决方案 |
|------|---------|
| 先写代码后补测试 | 删掉，用 TDD 重来 |
| 测试通过后停止 | 必须端到端验证 + eval |
| CGO 无法 mock | 用 HTTP 集成测试 |
| 忘记重启服务 | 新代码在 systemctl restart 前不生效 |
