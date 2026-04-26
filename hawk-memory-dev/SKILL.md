---
名称: hawk-memory-dev
描述: >
  hawk-memory Go 服务开发规范 — TDD 流程 + 测试 + eval 联动 + 服务管理。
  适用：任何在 hawk-memory 项目开发新功能或修复 bug。
版本: 1.0.0
作者: maomao
标签: [hawk-memory, Go, TDD, 测试, 开发]
---

# hawk-memory 开发规范

## 服务管理

**永远不要用 kill。** 用 systemd 管理服务：

```bash
# 重启
systemctl --user restart hawk-memory

# 查看状态
systemctl --user status hawk-memory

# 日志
journalctl --user -u hawk-memory -f

# 健康检查
curl http://localhost:18368/health
curl http://localhost:18368/v1/hooks/list
```

**DB 路径**：`/home/gql/.hawk/go-lancedb`（Go 服务专用，与 Python 版隔离）

## TDD 流程（Go + hawk-memory）

### 核心原则

```
RED  → 写失败的测试
GREEN → 写最简代码通过测试
REFACTOR → 清理
```

### hawk-memory 的测试策略

**HTTP API 集成测试为主**（推荐）
- hawk-memory 是长期运行的服务，无法在测试中启动 CGO 依赖
- 写 `*_test.go` 调用真实 HTTP 端点
- 测试前 `systemctl --user restart hawk-memory` 确保干净状态

**纯单元测试**（接口可 mock 时）
- 存储接口 `StorageBackend` 可以 mock
- 写 `*_test.go` 用 mock 实现

### RED — 写失败的测试

#### 1. 确定测试位置

```
pkg/xxx/          → 纯单元测试（mock 接口）
cmd/hawk-memory/  → 集成测试（HTTP 调用）
```

#### 2. 写集成测试模板

```go
// pkg/hooks/hooks_integration_test.go
package hooks

import (
	"bytes"
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"testing"
)

const baseURL = "http://localhost:18368"

func setup(t *testing.T) *httptest.Server {
	// 确保服务运行
	resp, err := http.Get(baseURL + "/health")
	if err != nil || resp.StatusCode != 200 {
		t.Skip("hawk-memory not running")
	}
	return nil
}

func TestHookRegister(t *testing.T) {
	// 先写测试：期望的请求和响应
	req := HookRegisterRequest{
		Name:        "test-hook",
		Description: "测试",
		Platforms:   []string{"openclaw"},
		Tags:        []string{"test"},
	}
	body, _ := json.Marshal(req)

	resp, err := http.Post(baseURL+"/v1/hooks/register", "application/json", bytes.NewReader(body))
	if err != nil {
		t.Fatalf("request failed: %v", err)
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		t.Errorf("expected 200, got %d", resp.StatusCode)
	}

	var result HookRegisterResponse
	json.NewDecoder(resp.Body).Decode(&result)
	if result.ID == "" {
		t.Error("expected non-empty hook ID")
	}
}
```

#### 3. 运行测试验证失败

```bash
cd ~/repos/hawk-memory
go test ./pkg/hooks/... -v -run TestHookRegister
# 预期：编译失败（接口未实现）或 HTTP 404（路由未注册）
```

### GREEN — 最少代码

实现功能，让测试通过。**不要添加测试未覆盖的功能。**

### REFACTOR — 清理

只在测试 green 后：
- 消除重复
- 改进命名
- 提取辅助函数

### 完整循环

```
1. 写测试文件（RED）
2. go test → 失败（功能未实现）
3. 实现功能（GREEN）
4. go test → 通过
5. go build ./cmd/hawk-memory → 编译成功
6. systemctl --user restart hawk-memory
7. curl 验证端到端
8. git commit
```

## 测试命令

```bash
# 单元测试（包内）
go test ./pkg/hooks/... -v

# 集成测试（HTTP）
go test ./cmd/... -v -tags=integration

# 所有测试
go test ./... -v

# 带覆盖率
go test ./... -cover

# 压力测试（decay）
go test ./internal/decay/... -bench=. -benchmem
```

## eval 联动

### 开发流程

```
1. 实现功能 → 单元测试通过
2. 重启服务（确保新代码生效）
3. 运行 eval 验证 recall 质量
   cd ~/repos/hawk-eval
   make benchmark-hawk
4. 检查 MRR/Recall/BLEU 指标
```

### 干净状态保障

每次 eval 前清理 DB：

```bash
# 停止服务
systemctl --user stop hawk-memory

# 清理 DB
rm -rf /home/gql/.hawk/go-lancedb/

# 重启（自动重建）
systemctl --user start hawk-memory

# 等待就绪
sleep 3
curl http://localhost:18368/health
```

详见 `hawk-benchmark-clean-state` skill。

## Go 代码规范

### 接口设计

先定义接口，再实现：

```go
// internal/storage/lancedb.go
type StorageBackend interface {
	Close() error
	Ping(ctx context.Context) error
	Capture(ctx context.Context, rec *MemoryRecord) (string, error)
	// 新方法加这里
	NewMethod(ctx context.Context, arg string) (Result, error)
}
```

然后实现 `NativeClient`（Go 存储）和 `Client`（HTTP 代理，Python 后端）。

### 提交规范

```
feat: KR3.6-5 Memory ROI API
fix: KR3.6-3 Decay batch sync race condition
test: KR3.6-2 Trust Score unit tests
docs: update API docs for /v1/analytics/roi
```

### 错误处理

- 用 `fmt.Errorf("operation: %w", err)` 包装错误
- HTTP handler 返回结构化错误：`c.JSON(http.StatusBadRequest, gin.H{"error": "message"})`
- 不要忽略 `err`

## 常见陷阱

| 陷阱 | 解决方案 |
|------|---------|
| CGO 依赖导致测试无法运行 | 用 HTTP 集成测试替代纯单元测试 |
| 先写代码后补测试 | 删掉实现，用 TDD 重来 |
| 测试通过后停止 | 必须验证端到端 + eval |
| 忘记重启服务 | 新代码在 systemctl restart 之前不生效 |
| DB 路径混用 | Go 服务用 `~/.hawk/go-lancedb`，Python 用 `~/.hawk/lancedb/` |
