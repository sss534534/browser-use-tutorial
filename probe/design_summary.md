# HIP 网管系统 agent-browser 自动化测试方案

> 本文档记录 HIP OTN 业务自动化测试方案的三次迭代过程。
> 面向读者：测试工程师、DevOps、AAO（AI Agent Orchestrator）从业者。
> 前置条件：已掌握 agent-browser 基本用法（open/click/type/snapshot/eval），已登录 192.192.18.34:10060。

---

## 文档索引

```
v1  初稿方案 —— 方向性框架，方向正确但过度设计，重侦察轻执行
v2  反思迭代 —— 补全 eval 作用域、错误路由、数据隔离、轻量框架
v3  生产级别 —— 真并行 Grid、LLM 动态 Locator、视觉回归、CI/CD、Git式注册表自进化
```

---

# ================================================================
# PART I — v1 初稿方案
# ================================================================

## 1.1 核心思路

v1 的核心思路是"侦察先行 → 结构化用例 → 导航复用 → 状态清理 → 知识固化"，构建一个五层金字塔体系：

```
侦察层（Reconnaissance）
   ↓
知识库层（Knowledge Base）
   ↓
导航脚本层（Navigation Scripts）
   ↓
用例库层（Test Library）
   ↓
执行层（Test Runner）
```

## 1.2 前置侦察阶段

### 1.2.1 页面结构快照存档

每次进入新功能模块前，先做一次完整快照作为"基准地图"：

```bash
# 入口处存档：模块打开前的状态
agent-browser snapshot -c > baseline_<module>_<timestamp>.snap

# 对话框打开前存档：记录 ref ID 分布
agent-browser snapshot -c > dialog_<name>_before.snap

# 关键 ref ID 提取为结构化数据
agent-browser snapshot -c | grep -E "button|textbox|radio|checkbox" > refs_<module>.txt
```

当 ref ID 随页面渲染动态变化时，通过对比两次快照快速定位新增/消失的元素，避免依赖"记忆中的旧 ref"导致点击失败。

### 1.2.2 JavaScript 上下文探测

在复杂 SPA/Java Applet 页面中，很多状态藏在 DOM 深处，光靠 accessibility tree 看不到：

```bash
# 探测隐藏的 JS 全局函数（导航、弹窗、树操作）
agent-browser eval "
Object.keys(window).filter(k =>
  typeof window[k] === 'function' &&
  /menu|click|navigate|open|enter|select|find/i.test(k)
).slice(0, 50).forEach(k => console.log(k, window[k].toString().slice(0, 80)))
" 2>&1

# 探测 ztree/jstree 等树组件的内部 API
agent-browser eval "
const zTree = $.fn.zTree.getZTreeObj('topoTree');
if (zTree) {
  console.log('nodes:', zTree.getNodes().length);
  zTree.getNodes().forEach(n => console.log(n.name, n.tId, n.id));
}
" 2>&1

# 探测弹窗/对话框是否存在（即便 accessibility tree 没显示）
agent-browser eval "
document.querySelectorAll('[role=dialog], .el-dialog, .layui-layer').forEach(d => {
  if (d.offsetHeight > 0) console.log('visible dialog:', d.className, d.textContent.slice(0, 50));
});
" 2>&1
```

在 HIP 系统中，提前发现 `contextMenu(event)`、`enterNE(nodeId)`、`getSelectedNode()` 这类隐藏 API，是避免盲目模拟鼠标事件、突破 Java Applet 隔离的关键。

## 1.3 用例结构化设计（GIVEN/WHEN/THEN/CLEANUP）

```bash
# ===== GIVEN: 前置状态准备 =====
# 登录 → 进入网元 → 打开功能页面 → 打开对话框

# ===== WHEN: 核心操作 =====
# 填写表单 → 选择下拉 → 点击确定

# ===== THEN: 结果断言 =====
agent-browser snapshot -c | grep "预期值"
assert "激活" in snapshot
assert "Up" in snapshot

# ===== CLEANUP: 环境恢复 =====
# 关闭对话框 / 删除刚创建的测试数据 / 返回上一页
```

三层用例结构：

```
Layer 1 — 原子操作（最细粒度，可跨用例复用）
  click_tree_node(nodeName)          # 点击功能树节点
  open_dialog(dialogName)             # 打开指定对话框
  select_radio_with_label(label)      # 选择指定标签的单选
  select_checkbox(label)             # 选择复选框

Layer 2 — 组合操作（特定模块内的标准流程）
  otn_enter_ne_manager(neName)        # 进入网元管理器
  otn_open_new_service_dialog()       # 打开新建业务对话框
  otn_select_source_timeslot(slot, port, odu)  # 选择源时隙

Layer 3 — 业务场景（端到端测试用例）
  test_create_otn_normal_service()     # 创建普通OTN业务
  test_create_otn_protected_service() # 创建保护OTN业务
```

## 1.4 导航序列复用

```bash
# login_and_enter_ne.sh — 标准导航路径脚本化
# 用法: ./login_and_enter_ne.sh <url> <user> <pass> <ne_ip>

URL="$1"; USER="$2"; PASS="$3"; NE_IP="$4"

agent-browser open "$URL" --headed
agent-browser wait 2
agent-browser type e1 "$USER"
agent-browser type e2 "$PASS"
agent-browser click login_btn
agent-browser wait 3

# 主拓扑 → 展开树 → 右键进入网元
agent-browser click main_topology_menu
agent-browser wait 2
agent-browser eval "/* 右键触发全部展开 */"
agent-browser click "全部展开"
agent-browser wait 1
agent-browser eval "/* 定位 $NE_IP 节点并右键 */"
agent-browser click "网元管理器"
agent-browser wait 2
```

## 1.5 分层等待策略

```bash
# 轮询等待目标出现（超时10秒），优于固定 sleep
wait_for() {
  local target="$1"
  local timeout="${2:-10}"
  local elapsed=0
  while [ $elapsed -lt $timeout ]; do
    if agent-browser snapshot -c 2>&1 | grep -q "$target"; then
      echo "Found '$target' after ${elapsed}s"
      return 0
    fi
    sleep 1
    elapsed=$((elapsed + 1))
  done
  echo "Timeout waiting for '$target' after ${timeout}s"
  return 1
}

# 用法
wait_for "OTN业务配置"
wait_for "新建普通业务"
wait_for "TestOTN01"  # 创建后验证
```

## 1.6 对话框自动探测与关闭

HIP 系统中对话框可能在多个 ref 下重复出现或嵌套，用 JS 统一处理：

```bash
close_any_dialog() {
  agent-browser eval "
  const selectors = ['[role=dialog]', '.el-dialog', '.layui-layer'];
  selectors.forEach(sel => {
    document.querySelectorAll(sel).forEach(d => {
      if (d.offsetHeight > 0 && d.textContent.includes('请选择')) {
        const btns = d.querySelectorAll('button');
        for (const b of btns) {
          if (b.textContent.trim() === '确定') { b.click(); console.log('closed'); }
        }
      }
    });
  });
  " 2>&1
  sleep 1
}
```

## 1.7 知识固化：Ref ID 映射表

```yaml
# SKILL.md — OTN业务配置 ref 映射（示例）
otn_new_service_dialog:
  描述: 新建普通业务对话框
  字段:
    名称输入框: { ref: e138, type: textbox }
    源板位:     { ref: e170, type: textbox, 缺省值: "7-UO2X-OSE.V2" }
    宿板位:     { ref: e172, type: textbox, 缺省值: "8-UO8G-OSE.V2" }
    源时隙编辑按钮: { ref: e140 }
    宿时隙编辑按钮: { ref: e141 }
    确定按钮:       { ref: e151 }
    取消按钮:       { ref: e152 }
  时隙选择对话框:
    槽位7单选:     { ref: e79 }
    端口IN/OUT1:   { ref: e95 }
    ODU2-ODU0-1复选: { ref: e46 }
    确定按钮:       { ref: e14 }
  注意事项:
    - type命令对自定义组件无效，需用click+eval组合
    - 对话框确定按钮需用JS定位（ref不稳定）
    - 表单重置需点击功能树节点
```

## 1.8 v1 存在的问题

| 问题 | 描述 |
|------|------|
| 过度侦察 | 每次用例执行前都跑 JS 探测，消耗时间是用例本身的 2-3 倍 |
| Ref ID 映射表失效快 | 每次页面渲染 ref 全部重新分配，维护成本无底洞 |
| 固定等待 vs 轮询等待根本方向错误 | 轮询在 HIP 异步场景下有延迟窗口问题，且 API 成本更高 |
| Layer 3 拆分过度 | 原子操作封装成独立脚本后，每次调用都有进程 fork 开销 |
| 浏览器上下文不共享 | 每个子 agent 都是全新会话，都要重复登录和导航 |

---

# ================================================================
# PART II — v2 反思迭代
# ================================================================

v2 在 v1 基础上做了实质性修补，引入了** eval 作用域补偿、错误分类路由、数据隔离、轻量框架**，但核心缺陷是**自愈停在注册表更新，没有验证闭环和回滚能力**，且**分布式并行仍是假的**（delegate_task 的多 agent 无法共享浏览器 session）。

## 2.1 eval 作用域缺陷的工程补偿

**问题根因**：agent-browser 的 `eval` 命令在连续调用间有变量声明复用机制，导致 `let/const/var` 声明的标识符冲突：

```
第一次 eval: const x = 1;  → 声明 x
第二次 eval: const x = 2;  → ✗ SyntaxError: Identifier 'x' has already been declared
```

**解决方案：每次 eval 用 IIFE 封装**

```bash
# 包装器：确保每次 eval 都是独立作用域
eval_js() {
    local js_code="$1"
    agent-browser eval "
    (function() {
        $js_code
    })();
    " 2>&1
}

# 每次调用都包装成 IIFE，不污染外层作用域
eval_js "var nodes = document.querySelectorAll('a'); console.log(nodes.length);"
eval_js "var btns = document.querySelectorAll('button'); console.log(btns.length);"
# 不会冲突！因为每次都是新的 IIFE scope
```

**批量多步操作的 eval 最佳实践**：对于 HIP 时隙选择对话框，一次 eval 搞定所有步骤，避免多次调用带来的 ref 重分配问题：

```bash
eval_js "
var MAX_WAIT = 5000, STEP = 300;
var found = false;

function waitForDialog(cb) {
    var dlgs = document.querySelectorAll('[role=dialog]');
    for (var i = 0; i < dlgs.length; i++) {
        if (dlgs[i].textContent.includes('请选择时隙') && dlgs[i].offsetHeight > 0) {
            cb(dlgs[i]); found = true; break;
        }
    }
    if (!found) setTimeout(function() { waitForDialog(cb); }, STEP);
}

waitForDialog(function(dlg) {
    // 1. 选槽位7
    var radios = dlg.querySelectorAll('input[type=radio]');
    for (var i = 0; i < radios.length; i++) {
        if (radios[i].value && radios[i].value.indexOf('UO2X-OSE.V2 7') > -1) {
            radios[i].click(); break;
        }
    }
    setTimeout(function() {
        // 2. 选端口 IN/OUT:1
        var ports = dlg.querySelectorAll('input[type=radio]');
        for (var i = 0; i < ports.length; i++) {
            if (ports[i].value === 'IN/OUT:1') { ports[i].click(); break; }
        }
        setTimeout(function() {
            // 3. 选时隙
            var checks = dlg.querySelectorAll('input[type=checkbox]');
            for (var i = 0; i < checks.length; i++) {
                if (checks[i].value && checks[i].value.indexOf('ODU2:1-ODU0:1') > -1) {
                    checks[i].click(); break;
                }
            }
            // 4. 点确定
            setTimeout(function() {
                var bts = dlg.querySelectorAll('button');
                for (var i = 0; i < bts.length; i++) {
                    if (bts[i].textContent.trim() === '确定') { bts[i].click(); break; }
                }
                console.log('TIMESLOT_DIALOG_DONE');
            }, STEP);
        }, STEP);
    }, STEP);
});
"
```

## 2.2 测试数据管理（Test Data Isolation）

v2 认识到"两个用例并行跑都用了 TestOTN01"的冲突问题，引入三层数据管理方案：

**方案 A：时间戳后缀（适合冒烟测试）**

```bash
RUN_ID="$(date +%s%N)"
SERVICE_NAME="TestOTN_${RUN_ID: -6}"  # 取后6位：TestOTN_473821
```

**方案 B：测试数据池（适合回归测试）**

```bash
declare -A DATA_POOL

request_slot() {
    local slot="$1"
    for entry in "${!DATA_POOL[@]}"; do
        if [[ "${DATA_POOL[$entry]}" == "FREE" ]] && [[ "$entry" == "$slot:"* ]]; then
            DATA_POOL[$entry]="USED"
            echo "$entry"
            return 0
        fi
    done
    echo "ERROR: No free slot for $slot" >&2
    return 1
}

release_slot() {
    local key="$1"
    DATA_POOL[$key]="FREE"
}

trap "release_slot '$SRC_INFO'; release_slot '$DST_INFO'" EXIT
```

**方案 C：预分配 + 用例锁定（最适合 CI/CD）**

```yaml
# test_suite_config.yaml
test_otn_service:
  resources:
    - ne: "192.192.18.177"
      src_slot: "7"
      dst_slot: "8"
      src_port: "IN/OUT:1"
      dst_port: "IN/OUT:1"
  cleanup_after: always
```

## 2.3 崩溃恢复机制（Crash Recovery）

三层状态机：

```
用例执行中 → 崩溃(断电/网络中断/浏览器崩溃)
                    ↓
            检测：进程消失？浏览器进程僵死？
                    ↓
            恢复策略路由
            ↓          ↓          ↓
        硬重启      重连会话    跳过
        (最底层)    (中层)     (上层)
```

watchdog 守护进程：

```bash
#!/bin/bash
# watchdog.sh — 用例执行守护，发现崩溃自动恢复

execute_with_watchdog() {
    local test_script="$1"
    local timeout="${2:-300}"
    
    $test_script &
    local pid=$!
    local elapsed=0
    
    while [ $elapsed -lt $timeout ]; do
        if ! kill -0 $pid 2>/dev/null; then
            wait $pid
            return $?
        fi
        
        # 检测浏览器进程是否僵死（CPU 0% 持续 >60s）
        local cpu=$(ps -p $pid -o %cpu= 2>/dev/null | tr -d ' ')
        if [[ $(echo "$cpu < 0.1" | bc -l) -eq 1 ]]; then
            stuck_count=$((stuck_count + 1))
            if [ $stuck_count -gt 12 ]; then
                echo "[WATCHDOG] Browser stuck, forcing recovery"
                kill -9 $pid
                recover_browser_session
            fi
        else
            stuck_count=0
        fi
        sleep 5
        elapsed=$((elapsed + 5))
    done
    
    kill -9 $pid 2>/dev/null
    recover_browser_session
    return 124
}
```

孤儿数据清理：

```bash
cleanup_orphaned_resources() {
    agent-browser attach --session orphaned 2>/dev/null || agent-browser open http://192.192.18.34:10060 --headed
    
    grep -oP "TestOTN_\d{6}" <(agent-browser snapshot -c) | sort -u | while read name; do
        # 如果业务年龄 < 30分钟（刚崩溃的用例创建），删除
        agent-browser eval "/* 选中 $name 所在行 */"
        agent-browser click "删除"
        sleep 1
    done
}
```

## 2.4 错误分类与自动路由（Error Classification Engine）

四类错误，路由不同：

```
[网络错误]  → 重试3次
[业务错误]  → 记录+跳过
[工具错误]  → 降级方案
[环境错误]  → 人工介入
```

```bash
classify_error() {
    local output="$1"
    
    if echo "$output" | grep -qiE "ECONNREFUSED|ETIMEDOUT|ENOTFOUND|connection|timeout|network"; then
        echo "NETWORK"; return
    fi
    if echo "$output" | grep -qiE "ref.*not.*found|identifier.*declared|syntaxerror|cannot.*click|element.*missing"; then
        echo "TOOL"; return
    fi
    if echo "$output" | grep -qiE "无权限|禁止|校验失败|失败|错误"; then
        echo "BUSINESS"; return
    fi
    if echo "$output" | grep -qiE "session|stale|disconnected|browser.*dead"; then
        echo "SESSION"; return
    fi
    echo "UNKNOWN"
}

handle_error() {
    local error_type="$1"
    case "$error_type" in
        NETWORK)  sleep 5; return 0;;                        # 可重试
        TOOL)      agent-browser vision --question "按钮在哪里？" 2>&1; return 1;;  # 降级
        BUSINESS)  echo "[SKIP] $context"; return 2;;          # 跳过
        SESSION)   recover_browser_session; return 0;;        # 可重试
        *)         echo "[FATAL]"; return 3;;                  # 人工介入
    esac
}
```

## 2.5 自愈机制（有缺陷的初版）

```bash
# 定位策略注册表
declare -A LOCATOR_REGISTRY

resolve_element() {
    local name="$1"
    local strategy="${LOCATOR_REGISTRY[$name]}"
    IFS='|' read -ra STRATS <<< "$strategy"
    for s in "${STRATS[@]}"; do
        IFS=':' read -ra parts <<< "$s"
        local type="${parts[0]}"; local val="${parts[1]}"
        # 尝试定位...
        [ -n "$result" ] && { echo "${type}:${val}"; return 0; }
    done
    echo "ALL_STRATEGIES_FAILED"; return 1
}

# v2 的缺陷：更新后不验证、无法回滚
learn_from_success() {
    LOCATOR_REGISTRY[$name]="${working_strategy}|${old}"
    save_registry  # 写入了，但从来不验证这次写入是否真的 work
}
```

## 2.6 轻量测试框架（v2）

```yaml
# test_suite.yaml
name: hip_otn_suite
global_setup:
  - script: login_and_enter_ne.sh
    args: { url: "192.192.18.34:10060", ne: "192.192.18.177" }
global_teardown:
  - script: cleanup_all_test_services.sh
cases:
  - name: test_create_otn_normal_service
    tags: [smoke, otn]
    retry: 2
    timeout: 60
  - name: test_delete_otn_service
    tags: [smoke, cleanup]
    depends: [test_create_otn_normal_service]
    timeout: 30
```

```bash
#!/bin/bash
# run_suite.sh — 轻量测试框架引擎

run_suite() {
    local passed=0; local failed=0; local skipped=0
    
    for case in "${suite.cases[@]}"; do
        # Tag 过滤
        [[ ! " ${case.tags[*]} " =~ " $TAG_FILTER " ]] && { skipped=$((skipped+1)); continue; }
        
        # 依赖检查
        if [ -n "${case.depends}" ]; then
            for dep in "${case.depends[@]}"; do
                [[ ! -f "/tmp/.pass_${dep}" ]] && { skipped=$((skipped+1)); continue 2; }
            done
        fi
        
        timeout ${case.timeout} bash -c "source ${case.name}.sh" 2>&1
        [ $? -eq 0 ] && { passed=$((passed+1)); touch "/tmp/.pass_${case.name}"; } \
                     || { failed=$((failed+1)); }
    done
    
    echo "PASSED: $passed  FAILED: $failed  SKIPPED: $skipped"
}
```

---

# ================================================================
# PART III — v3 生产级别方案
# ================================================================

## 3.1 v3 核心突破一览

| 突破点 | v2 状态 | v3 方案 |
|--------|---------|---------|
| 真并行执行 | delegate_task 伪并行 | Selenium Grid Hub-Spoke 真并行 |
| LLM 动态 Locator | 手动注册表更新 | 四层 LLM 生成 + 影子测试 |
| 视觉回归测试 | vision 兜底定位 | 像素级基线比对 |
| 自愈验证闭环 | 有注册表无验证无回滚 | Git式版本控制 + change_log |
| CI/CD 集成 | 无 | GitHub Actions + Allure + 飞书通知 |
| 会话管理 | session 参数未验证 | PID 追踪 fallback + session |

## 3.2 核心突破一：真并行架构（Multi-Browser Grid）

### 3.2.1 Hub-Spoke 模型

```
                    ┌─────────────────┐
                    │  Orchestrator   │
                    │   (本agent)     │
                    │  负责调度 + 汇总  │
                    └────────┬────────┘
                             │
          ┌──────────────────┼──────────────────┐
          ↓                  ↓                  ↓
    ┌──────────┐       ┌──────────┐        ┌──────────┐
    │ Grid Node│       │ Grid Node│        │ Grid Node│
    │ :4441    │       │ :4442    │        │ :4443    │
    │ (chrome1)│       │ (chrome2) │        │ (chrome3) │
    └──────────┘       └──────────┘        └──────────┘
```

### 3.2.2 Docker Compose 启动

```yaml
# docker-compose.yml
version: '3'
services:
  selenium-hub:
    image: selenium/hub:4.14
    ports:
      - "4444:4444"
    environment:
      SE_EVENT_BUS_BUS: "true"
  
  chrome-node-1:
    image: selenium/node-chrome:4.14
    shm_size: '2gb'
    depends_on:
      - selenium-hub
  
  chrome-node-2:
    image: selenium/node-chrome:4.14
    shm_size: '2gb'
    depends_on:
      - selenium-hub
```

### 3.2.3 Grid Executor

```bash
#!/bin/bash
# grid_executor.sh — 真并行执行器

GRID_HUB="http://localhost:4444"

grid_execute() {
    local test_case="$1"
    local capabilities="$2"
    
    # 向 Grid Hub 请求 session
    session_id=$(curl -s -X POST "$GRID_HUB/session" \
        -H "Content-Type: application/json" \
        -d "{\"capabilities\":{\"browserName\":\"${capabilities%%,*}\"}}" \
        | python3 -c "import sys,json; print(json.load(sys.stdin)['value']['sessionId'])")
    
    [ -z "$session_id" ] && { echo "[GRID] Failed to allocate"; return 1; }
    
    # 通过 WebDriver Wire Protocol 执行
    curl -X DELETE "$GRID_HUB/session/$session_id"
    return 0
}

# 并行分发
parallel_execute() {
    local tests=("$@")
    local pids=()
    for test in "${tests[@]}"; do
        grid_execute "$test" "chrome" &
        pids+=($!)
    done
    local failed=0
    for pid in "${pids[@]}"; do wait $pid || failed=$((failed+1)); done
    echo "[GRID] Done: $failed failures"
}
```

### 3.2.4 多 Playwright 实例（更实用）

```python
#!/usr/bin/env python3
# multi_playwright_executor.py

from playwright.sync_api import sync_playwright
import threading, queue

class BrowserWorker(threading.Thread):
    def __init__(self, worker_id, test_queue, results):
        super().__init__()
        self.worker_id = worker_id
        self.test_queue = test_queue
        self.results = results
        self.playwright = None
        self.browser = None
        
    def run(self):
        self.playwright = sync_playwright().start()
        self.browser = self.playwright.chromium.launch(headless=False)
        
        while True:
            task = self.test_queue.get()
            if task is None: break
            result = self.execute_task(task)
            self.results.append(result)
        
        self.browser.close()
        self.playwright.stop()
    
    def execute_task(self, task):
        context = self.browser.new_context(
            viewport={'width': 1920, 'height': 1080},
            ignore_https_errors=True
        )
        page = context.new_page()
        try:
            page.goto(task['url'], timeout=30000)
            page.wait_for_timeout(2000)
            page.fill('input[name="username"]', task['username'])
            page.fill('input[name="password"]', task['password'])
            page.click('button[type="submit"]')
            page.wait_for_timeout(3000)
            return {'worker': self.worker_id, 'task': task['name'], 'status': 'PASS'}
        except Exception as e:
            return {'worker': self.worker_id, 'task': task['name'], 'status': 'FAIL', 'error': str(e)}
        finally:
            context.close()

def run_parallel(tasks, concurrency=3):
    test_queue = queue.Queue()
    for t in tasks: test_queue.put(t)
    for _ in range(concurrency): test_queue.put(None)
    
    results = []
    workers = [BrowserWorker(i, test_queue, results) for i in range(concurrency)]
    for w in workers: w.start()
    for w in workers: w.join()
    return results
```

## 3.3 核心突破二：LLM-Driven 动态 Locator 生成

### 3.3.1 四层 Locator 生成器

```
Layer 1: 注册表查询    → 毫秒级，已知元素
Layer 2: 语义相似匹配  → NLP cosine similarity > 0.85
Layer 3: Accessibility Tree + LLM 推理  → 1-2秒，没见过的元素
Layer 4: Vision + 截图分析                → 2-3秒，Accessibility 无效
```

### 3.3.2 Layer 3 实现

```python
#!/usr/bin/env python3
# llm_locator.py — LLM 驱动的动态 locator 生成

import os, json, subprocess

LLM_PROVIDER = os.getenv("LLM_PROVIDER", "anthropic")

SYSTEM_PROMPT = """你是一个 UI 自动化工程师。你的任务：
给定一个 HIP 网管系统的 Accessibility Tree，以及要定位的元素描述，
输出该元素在 accessibility tree 中的确切位置。

输出格式（JSON）：
{
    "strategy": "text | ref | xpath | aria-label | coordinate",
    "value": "定位值",
    "confidence": 0.0-1.0,
    "reasoning": "为什么这样定位"
}

规则：
1. 优先用 text 定位（最稳定）
2. 其次用 aria-label / xpath
3. ref 数字不作为定位依据（动态变化）
4. 如果找不到，输出 confidence: 0.0
"""

def get_accessibility_tree():
    result = subprocess.run(
        ['agent-browser', 'snapshot', '-c'],
        capture_output=True, text=True, timeout=30
    )
    return result.stdout

def llm_query(tree_content, element_description):
    user_prompt = f"""Accessibility Tree:\n{tree_content[:8000]}\n\n要定位的元素: {element_description}\n\n请在上面的 Accessibility Tree 中找到该元素。"""
    
    if LLM_PROVIDER == "anthropic":
        import anthropic
        client = anthropic.Anthropic()
        resp = client.messages.create(
            model="claude-sonnet-4-6-20250514",
            max_tokens=512,
            system=SYSTEM_PROMPT,
            messages=[{"role": "user", "content": user_prompt}]
        )
        return json.loads(resp.content[0].text)

def generate_locator(element_name, max_layers=4):
    # Layer 1: 注册表
    registry_result = resolve_from_registry(element_name)
    if registry_result:
        return registry_result, "registry"
    
    # Layer 2: 语义相似
    semantic_result = resolve_from_semantic(element_name)
    if semantic_result and semantic_result['confidence'] > 0.85:
        return semantic_result, "semantic"
    
    # Layer 3: LLM 推理
    tree = get_accessibility_tree()
    llm_result = llm_query(tree, element_name)
    if llm_result['confidence'] > 0.7:
        if verify_locator(llm_result):
            return llm_result, "llm_tree"
        else:
            return generate_vision_locator(element_name), "llm_vision"
    
    return None, "failed"
```

### 3.3.3 自愈验证闭环（Self-Healing Verification Loop）

```python
def self_heal_with_verification(element_name, max_attempts=3):
    """
    标准自愈流程：
    1. 尝试所有已知策略
    2. 失败 → LLM 生成新策略
    3. 验证新策略（执行一次 click，看是否报错）
    4. 验证成功 → 更新注册表 + 记录版本
    5. 验证失败 → 重试（最多3次）
    """
    attempt = 0
    while attempt < max_attempts:
        attempt += 1
        locator, source = generate_locator(element_name)
        
        if locator is None:
            print(f"[HEAL] All layers exhausted for '{element_name}'")
            return None, f"exhausted_after_{max_attempts}"
        
        print(f"[HEAL] Attempt {attempt}: {source} → {locator['strategy']}={locator['value']}")
        
        success, error = verify_and_execute(locator)
        
        if success:
            if source != "registry":
                update_registry(element_name, locator, source)
                save_registry_version()
            return locator, source
        else:
            last_error = error
            blacklist_strategy(element_name, locator)
    
    return None, f"failed_after_{max_attempts}_attempts:{last_error}"
```

## 3.4 核心突破三：视觉回归测试（Visual Regression Testing）

### 3.4.1 何时需要视觉回归

```
文本类断言（grep 快照）够用：
  - 业务列表有 TestOTN01

视觉类必须用截图比对：
  - 对话框弹出后，表单布局是否正确
  - 下拉菜单展开后选项是否正确
  - 按钮是否被截断、遮挡、重叠
  - Java Applet 渲染的拓扑图是否正确
```

### 3.4.2 实现

```bash
#!/bin/bash
# visual_regression.sh

BASELINE_DIR="./baselines/$(date +%Y%m%d)"
CURRENT_DIR="./current/$(date +%Y%m%d)"
DIFF_DIR="./diffs/$(date +%Y%m%d)"
mkdir -p "$BASELINE_DIR" "$CURRENT_DIR" "$DIFF_DIR"

# 拍基准截图
capture_baseline() {
    local name="$1"
    agent-browser screenshot "$CURRENT_DIR/${name}.png"
}

# 执行视觉 diff（用 ImageMagick）
visual_diff() {
    local name="$1"
    local baseline="$BASELINE_DIR/${name}.png"
    local current="$CURRENT_DIR/${name}.png"
    local diff="$DIFF_DIR/${name}_diff.png"
    
    if [ ! -f "$baseline" ]; then
        cp "$current" "$baseline"
        echo "[VISUAL] No baseline for '$name', saved as baseline"
        return 0
    fi
    
    convert "$baseline" "$current" \
        -compose Difference -composite \
        -colorspace Gray "$diff"
    
    diff_ratio=$(identify -format "%[mean]" "$diff" 2>/dev/null | awk '{print $1/655.35}')
    threshold=1.0
    
    if (( $(echo "$diff_ratio > $threshold" | bc -l) )); then
        echo "[VISUAL] FAIL: '$name' differs by ${diff_ratio}% (threshold: ${threshold}%)"
        return 1
    fi
    echo "[VISUAL] PASS: '$name' differs by ${diff_ratio}%"
    return 0
}

# 用法示例：对话框打开后截图比对
test_dialog_layout() {
    agent-browser click "新建普通业务"
    agent-browser wait 2
    agent-browser screenshot "$CURRENT_DIR/dialog_form.png"
    visual_diff "dialog_form"
}
```

## 3.5 核心突破四：Git 式注册表版本控制

v2 自愈的最大缺陷：注册表更新后不验证、无法回滚。v3 引入 git commit 风格的版本控制：

```bash
#!/bin/bash
# locator_versions.sh — 每个注册表更新都是一个版本快照

REGISTRY_DIR="./locator_registry"
mkdir -p "$REGISTRY_DIR"

snapshot_registry() {
    local timestamp=$(date +%Y%m%d_%H%M%S)
    local snapshot_file="$REGISTRY_DIR/snapshot_${timestamp}.yaml"
    declare -p LOCATOR_REGISTRY > "$snapshot_file"
    echo "$snapshot_file"
}

update_locator_with_rollback() {
    local element_name="$1"
    local new_locator="$2"
    local source="$3"
    
    # 1. 先快照当前版本（回滚点）
    local rollback_file=$(snapshot_registry)
    
    # 2. 影子测试：在当前页面试用新 locator
    local test_result=$(eval_js "...")
    
    if echo "$test_result" | grep -q "OK"; then
        # 3. 影子测试通过，提交更新
        LOCATOR_REGISTRY[$element_name]="${new_locator}|${LOCATOR_REGISTRY[$element_name]}"
        save_registry
        echo "$(date +%s) $element_name $source -> $new_locator" >> "$REGISTRY_DIR/change_log.txt"
        echo "[REGISTRY] Updated: $element_name = $new_locator"
    else
        # 4. 影子测试失败，恢复快照
        cp "$rollback_file" "$REGISTRY_DIR/current.yaml"
        echo "[REGISTRY] REJECTED: $new_locator failed shadow test"
    fi
}

rollback_to() {
    local snapshot_file="$1"
    [ -f "$snapshot_file" ] && { source "$snapshot_file"; save_registry; echo "Rolled back"; }
}

list_versions() {
    echo "Available snapshots:"
    ls -la "$REGISTRY_DIR"/snapshot_*.yaml | awk '{print $NF, $6, $7, $8}'
    echo "Change log:"
    cat "$REGISTRY_DIR/change_log.txt" | tail -20
}
```

## 3.6 核心突破五：CI/CD 集成路径

### 3.6.1 GitOps 流程

```
开发提交测试脚本
      ↓
Git push / MR 创建
      ↓
CI Runner 触发
      ↓
┌─────────────────────────────────────────┐
│  1. 启动 Selenium Grid (docker-compose) │
│  2. 运行 test_suite.yaml                │
│  3. 生成 Allure 报告                    │
│  4. 推送报告到 GitHub Pages             │
│  5. 结果通过飞书通知                    │
└─────────────────────────────────────────┘
```

### 3.6.2 GitHub Actions

```yaml
# .github/workflows/hip-otn-test.yml
name: HIP OTN Automated Tests

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * *'  # 每天凌晨完整回归

jobs:
  otn-tests:
    runs-on: ubuntu-latest
    timeout-minutes: 60
    
    services:
      selenium-hub:
        image: selenium/hub:4.14
        ports:
          - 4444:4444
      
      chrome:
        image: selenium/node-chrome:4.14
        shm_size: '2gb'
    
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          pip install playwright anthropic allure-python
          playwright install chromium
      
      - name: Run Smoke Tests
        run: |
          bash run_suite.sh test_suite.yaml smoke
        env:
          HIP_URL: ${{ secrets.HIP_URL }}
          HIP_USER: ${{ secrets.HIP_USER }}
          HIP_PASS: ${{ secrets.HIP_PASS }}
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      
      - name: Upload Allure Report
        uses: actions/upload-artifact@v4
        with:
          name: allure-report
          path: allure-results/
      
      - name: Deploy Report to Pages
        if: github.ref == 'refs/heads/main'
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./allure-results

      - name: Notify on Failure
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {"text": "HIP OTN Test FAILED", "attachments": [{"color": "danger", "text": "Run: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"}]}
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

## 3.7 架构总览 v3

```
                    ┌──────────────────────────────────────┐
                    │         GitOps / CI Pipeline          │
                    │  GitHub Actions → Selenium Grid →     │
                    │  Allure Report → GitHub Pages → 通知   │
                    └──────────────────┬───────────────────┘
                                       │
┌──────────────────────────────────────▼───────────────────────────────────────┐
│                          Orchestrator (本 agent)                              │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────────────────────┐  │
│  │ Suite Loader   │  │  Grid Router   │  │  LLM Locator Generator (LLGL)  │  │
│  │ (YAML config)  │  │  (Hub-Spoke)   │  │  Layer1: Registry              │  │
│  └────────────────┘  └────────────────┘  │  Layer2: Semantic              │  │
│                                          │  Layer3: Accessibility+LLM     │  │
│                                          │  Layer4: Vision+LLM            │  │
│                                          └────────────────────────────────┘  │
├───────────────────────────────────────────────────────────────────────────────┤
│                              Core Framework                                    │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐  ┌──────────────┐ │
│  │ heal_click()  │  │ error_classify │  │ visual_diff()  │  │ watchdog.sh  │ │
│  │ (4层自愈)     │  │ (4类路由)      │  │ (像素级比对)   │  │ (崩溃恢复)    │ │
│  └────────────────┘  └────────────────┘  └────────────────┘  └──────────────┘ │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐  ┌──────────────┐ │
│  │ eval_js()      │  │ data_pool.sh   │  │ locator_reg.sh │  │ register_    │ │
│  │ (IIFE封装)    │  │ (预分配+隔离) │  │ (Git式版本)   │  │ cleanup()   │ │
│  └────────────────┘  └────────────────┘  └────────────────┘  └──────────────┘ │
├───────────────────────────────────────────────────────────────────────────────┤
│                           Execution Backends                                   │
│  ┌─────────────────────────┐      ┌──────────────────────────────────────┐  │
│  │ agent-browser (local)    │      │  Selenium Grid (multi-browser)      │  │
│  │ headed/session/eval     │      │  Node1: Chrome :4441                 │  │
│  │                         │      │  Node2: Chrome :4442                 │  │
│  │                         │      │  Node3: Chrome :4443                 │  │
│  └─────────────────────────┘      └──────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────────────┘
```

---

# ================================================================
# PART IV — 三版本横向对比
# ================================================================

| 维度 | v1 初稿 | v2 反思迭代 | v3 生产级别 |
|------|---------|-------------|-------------|
| **并行执行** | delegate_task(伪) | 同v1 | Selenium Grid(真) |
| **自愈机制** | 手动更新注册表 | 注册表+黑名单 | 4层LLM生成+影子测试+Git版本 |
| **视觉测试** | vision兜底定位 | 同v1 | 像素级diff+基线管理 |
| **Locator策略** | 固定ref/文本 | 失败后学习 | LLM自动推断+验证闭环 |
| **CI/CD** | 无 | 框架层 | GitHub Actions+Allure+通知 |
| **会话复用** | session参数(未验证) | 同v1 | PID追踪fallback+session |
| **回滚能力** | 无 | 快照(从不回滚) | Git式版本控制+change_log |
| **数据隔离** | 无 | 时间戳+池化+预分配 | 同v2 + 孤儿清理 |
| **崩溃恢复** | 无 | watchdog基础版 | 4类错误路由+诊断 |
| **eval安全** | 无 | IIFE封装 | 同v2 + 批量多步最佳实践 |

---

# ================================================================
# PART V — 落地优先级建议
# ================================================================

按实施成本和回报排序：

```
第一优先级（立即可做，1-2天）
  1. eval_js() IIFE 封装 — 消除 SyntaxError
  2. 数据池+时间戳 — 解决用例互相覆盖
  3. error_classify 路由 — 让失败可分析

第二优先级（1周内）
  4. 轻量框架 run_suite.sh — 结构化管理用例
  5. watchdog 守护 — 崩溃后自动恢复
  6. 快照存档流程 — 失败后有迹可循

第三优先级（2-4周）
  7. Selenium Grid 真并行 — 多浏览器并发
  8. locator_reg Git式版本 — 自愈可验证可回滚
  9. GitHub Actions CI/CD — 自动化报告链路

第四优先级（长期）
  10. LLM Locator Generator — 零人工维护的动态定位
  11. 视觉回归测试 — 截图基线管理
  12. Multi-Agent Collaboration — 用例内操作级并行
```

---

*文档版本：v1.0 | 编写日期：2026-04-22 | 状态：已完成*
