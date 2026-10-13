# Lab 25B — Sơ đồ Kiến trúc & Luồng Xử lý Hệ thống (UML Workflows)

Tài liệu này đặc tả toàn bộ các luồng nghiệp vụ (workflows), kiến trúc phân lớp, máy trạng thái (state machines), và cơ chế xử lý lỗi bền bỉ trong **Lab 25B: Circuit Breakers, Caching, Reliability & SRE Error Budgets**.

Mỗi quy trình được mô hình hóa bằng chuẩn **Mermaid UML** (hiển thị trực tiếp trong VS Code / GitHub Markdown) và đi kèm **PlantUML Source Code**.

---

## Mục Lục

1. [Tổng quan Kiến trúc Hệ thống (Component Architecture)](#1-tổng-quan-kiến-trúc-hệ-thống)
2. [Luồng Xử lý Yêu cầu Đa tầng Resilient LLM Routing (Sequence Diagram)](#2-luồng-xử-lý-yêu-cầu-đa-tầng-resilient-llm-routing)
3. [Máy trạng thái Circuit Breaker Cửa sổ Trượt (State Machine Diagram)](#3-máy-trạng-thái-circuit-breaker-cửa-sổ-trượt)
4. [Luồng Xử lý Workflow Hoàn tiền Chính xác Một Lần — Exactly-Once (Sequence Diagram)](#4-luồng-xử-lý-workflow-hoàn-tiền-chính-xác-một-lần--exactly-once)
5. [Cơ chế Phục hồi khi Gặp Crash & Idempotency (Activity Diagram)](#5-cơ-chế-phục-hồi-khi-gặp-crash--idempotency)
6. [Công cụ Cảnh báo SRE Multi-Window Multi-Burn-Rate (Sequence Diagram)](#6-công-cụ-cảnh-báo-sre-multi-window-multi-burn-rate)
7. [Sơ đồ Lớp Toàn diện (Class Diagram)](#7-sơ-đồ-lớp-toàn-diện)

---

## 1. Tổng quan Kiến trúc Hệ thống

Sơ đồ thể hiện sự phối hợp giữa các tầng: **Client Layer**, **Caching Layer**, **Resilience Layer (Retry/Breaker)**, **Durable Workflow Engine**, và **SRE Monitoring Engine**.

### Mermaid Diagram
```mermaid
graph TD
    User([Người dùng / Agent Task]) --> CachedClient[CachedClient\n(Semantic Cache)]
    
    subgraph "Caching Layer (Part G)"
        CachedClient -->|Cosine Similarity >= 0.92| CacheHit{Kiểm tra Cache?}
        CacheHit -->|Fresh: age <= 60s| ReturnCached([Trả lời từ Cache - 0ms, 0$])
        CacheHit -->|Miss / Stale| ResilientClient[ResilientClient\n(Multi-Provider Router)]
        CacheHit -->|Upstream All Fail & age <= 300s| ServeStale([Stale-If-Error Fallback])
    end

    subgraph "Resilience Layer (Parts A, B, C)"
        ResilientClient --> RetryBudgetCheck{Retry Budget\nToken Available?}
        RetryBudgetCheck -->|Yes| ProviderSelection[Chọn Provider theo Ladder:\nPrimary -> Secondary -> Fallback]
        RetryBudgetCheck -->|Exhausted| GiveUpBudget([GiveUp: retry_budget_exhausted])
        
        ProviderSelection --> BreakerCheck{Circuit Breaker\nState?}
        BreakerCheck -->|OPEN| Failover[Chuyển Provider tiếp theo]
        BreakerCheck -->|CLOSED / HALF-OPEN| DispatchProvider[Gửi yêu cầu tới Provider]
        
        DispatchProvider --> ErrorClassifier{Phân loại phản hồi\nErrorKind}
        ErrorClassifier -->|Success| RecordSuccess[Breaker.on_success\nBudget.record_success]
        ErrorClassifier -->|Rate-Limited / Overloaded| RetryWithJitter[FullJitterBackoff\nThử lại cùng Provider]
        ErrorClassifier -->|Timeout / 5xx Crash| BreakerFail[Breaker.on_failure\nFailover Provider tiếp]
    end

    subgraph "Durable Workflow Layer (Part D)"
        WorkflowTask([Yêu cầu Hoàn tiền RefundRequest]) --> WorkflowExec[Durable Workflow Executor]
        WorkflowExec <--> DurableStore[(DurableStore\nCheckpoints & Idempotency Keys)]
        WorkflowExec --> Step1[Step 1: lookup_policy]
        WorkflowExec --> Step2[Step 2: call_llm_judge]
        WorkflowExec --> Step3[Step 3: ledger_debit]
        WorkflowExec --> Step4[Step 4: gateway_refund]
        WorkflowExec --> Step5[Step 5: send_email]
        WorkflowExec --> Step6[Step 6: record_audit]
    end

    subgraph "SRE Alerting & SLO Engine (Part E)"
        TimeSeriesData[(Tập dữ liệu SLI 30 ngày\n43.200 phút)] --> WindowSums[WindowSums: O(1) Prefix Sums]
        WindowSums --> MultiBurnRate[Bộ đánh giá Multi-Burn-Rate]
        MultiBurnRate --> AlertEngine{Kiểm tra 4 Bậc Rule}
        AlertEngine -->|14.4x burn (1h & 5m)| PageUrgent([PAGER: Khẩn cấp - Mất 2% budget/1h])
        AlertEngine -->|6.0x burn (6h & 30m)| PageSevere([PAGER: Nghiêm trọng - Mất 5% budget/6h])
        AlertEngine -->|3.0x burn (24h & 2h)| TicketWarning([TICKET: Cảnh báo - Mất 10% budget/ngày])
        AlertEngine -->|1.0x burn (72h & 6h)| TicketTrend([TICKET: Xu hướng - Mất 10% budget/3 ngày])
    end
```

<details>
<summary><b>Nhấn để xem mã nguồn PlantUML</b></summary>

```plantuml
@startuml
skinparam componentStyle rectangle

package "Caching Layer (Part G)" {
    [CachedClient] as Cache
    database "FAQ Cache Store" as CacheStore
}

package "Resilience Gateway (Parts A, B, C)" {
    [ResilientClient] as Router
    [RetryBudget & Jitter] as Retry
    [SlidingWindowBreaker] as Breaker
}

cloud "LLM Providers" {
    [Primary: OpenAI] as P1
    [Secondary: Anthropic] as P2
    [Fallback: Local LLM] as P3
}

package "Durable Workflows (Part D)" {
    [WorkflowExecutor] as Executor
    database "DurableStore" as Store
}

package "SRE Observability (Part E)" {
    [WindowSums Engine] as SRE
    [Multi-Burn-Rate Alerter] as Alerter
}

actor User
User -> Cache : Submit Query
Cache -> CacheStore : Cosine Search (theta=0.92)
Cache --> User : Cache Hit (Fresh / Stale-if-error)
Cache -> Router : Cache Miss
Router -> Retry : Check Retry Token
Router -> Breaker : Acquire Permit
Breaker -> P1 : Call Primary
P1 ..> Breaker : Failure / Timeout
Router -> P2 : Failover Secondary
Router -> P3 : Failover Fallback

Executor <-> Store : Checkpoint Step & Key
Executor -> P1 : LLM Judge Step

SRE -> Alerter : 43,200 min Prefix Sums
Alerter -> User : Page / Ticket Alerts
@enduml
```
</details>

---

## 2. Luồng Xử lý Yêu cầu Đa tầng Resilient LLM Routing

Sơ đồ trình tự biểu diễn chi tiết cách một lệnh gọi LLM được bảo vệ bởi **Semantic Cache**, **Retry Budget**, **Circuit Breaker per-provider**, và **Fallback Ladder** khi nhà cung cấp chính gặp sự cố (Rate Limit, Quota, Timeout).

### Mermaid Diagram
```mermaid
sequenceDiagram
    autonumber
    actor App as Ứng dụng / Agent
    participant Cache as CachedClient
    participant Client as ResilientClient
    participant Budget as RetryBudget
    participant CB_Pri as Breaker (Primary)
    participant Primary as LLM Primary
    participant CB_Sec as Breaker (Secondary)
    participant Secondary as LLM Secondary

    App->>Cache: send_call(prompt, deadline)
    Note over Cache: Tính Cosine Similarity với FAQ keys
    
    alt Trường hợp Cache Hit (Tương đồng >= 0.92 & age <= 60s)
        Cache-->>App: Trả về kết quả Cache (HIT, 0ms latency)
    else Trường hợp Cache Miss hoặc Outdated
        Cache->>Client: forward_call(prompt, deadline)
        
        Note over Client: Chọn Provider đầu tiên trong Ladder: [Primary, Secondary, Fallback]
        Client->>CB_Pri: try_acquire(now)
        
        alt Circuit Breaker Primary đang CLOSED
            CB_Pri-->>Client: Permit được cấp
            Client->>Primary: Gửi Request (attempt 1)
            
            alt Primary trả về 429 Rate Limited hoặc 503 Overloaded
                Primary-->>Client: HTTP 429 (Retry-After: 1.5s)
                Client->>CB_Pri: on_failure(Permit, now)
                
                Note over Client: Kiểm tra số lần thử < max_attempts (2)\nvà deadline còn đủ
                Client->>Budget: try_acquire_retry(now)
                
                alt Retry Budget CÒN Token
                    Budget-->>Client: Cho phép thử lại
                    Note over Client: Tính delay = FullJitterBackoff(attempt)
                    Client->>Client: Chờ delay (thời gian ảo)
                    Client->>CB_Pri: try_acquire(now + delay)
                    CB_Pri-->>Client: Permit được cấp
                    Client->>Primary: Gửi Retry Request (attempt 2)
                    Primary-->>Client: Tiếp tục lỗi 503 / Timeout
                    Client->>CB_Pri: on_failure(Permit, now)
                else Retry Budget HẾT Token
                    Budget-->>Client: Từ chối retry (ngăn bão retry storm)
                end
                
                Note over Client: Thử lại thất bại -> Chuyển sang Secondary Provider
                Client->>CB_Sec: try_acquire(now)
                CB_Sec-->>Client: Permit được cấp
                Client->>Secondary: Gửi Request sang Secondary
                Secondary-->>Client: HTTP 200 OK (Thành công)
                Client->>CB_Sec: on_success(Permit, latency, now)
                Client->>Budget: record_success(now)
                Client-->>Cache: Trả về kết quả
                Cache->>Cache: Lưu vào Cache Store (prompt_emb, response, now)
                Cache-->>App: Hoàn tất phản hồi
                
            else Primary trả về 200 OK
                Primary-->>Client: HTTP 200 OK
                Client->>CB_Pri: on_success(Permit, latency, now)
                Client->>Budget: record_success(now)
                Client-->>Cache: Trả về kết quả
                Cache->>Cache: Lưu vào Cache Store
                Cache-->>App: Hoàn tất phản hồi
            end

        else Circuit Breaker Primary đang OPEN
            CB_Pri-->>Client: Từ chối (CircuitOpenError - Fail Fast)
            Note over Client: Bỏ qua Primary không gọi mạng,\nchuyển ngay sang Secondary
            Client->>CB_Sec: try_acquire(now)
            CB_Sec-->>Client: Permit được cấp
            Client->>Secondary: Gửi Request
            Secondary-->>Client: HTTP 200 OK
            Client-->>App: Phản hồi thành công
        end
    end
```

<details>
<summary><b>Nhấn để xem mã nguồn PlantUML</b></summary>

```plantuml
@startuml
autonumber
actor Caller
participant CachedClient
participant ResilientClient
participant RetryBudget
participant "Breaker Primary" as CBP
participant "Primary LLM" as P
participant "Breaker Secondary" as CBS
participant "Secondary LLM" as S

Caller -> CachedClient: send(query)
alt Semantic Cache Hit
    CachedClient --> Caller: Cached Response (Hit)
else Cache Miss
    CachedClient -> ResilientClient: route(call)
    ResilientClient -> CBP: try_acquire(now)
    alt Breaker Closed
        CBP --> ResilientClient: Permit
        ResilientClient -> P: HTTP Request
        P --> ResilientClient: 503 Service Unavailable
        ResilientClient -> CBP: on_failure(permit)
        ResilientClient -> RetryBudget: try_acquire_retry(now)
        alt Budget Granted & Time Left
            RetryBudget --> ResilientClient: OK
            ResilientClient -> P: Retry with Jitter
            P --> ResilientClient: Timeout
            ResilientClient -> CBP: on_failure(permit)
        end
        ResilientClient -> CBS: Failover to Secondary
        CBS --> ResilientClient: Permit
        ResilientClient -> S: HTTP Request
        S --> ResilientClient: 200 OK
        ResilientClient -> CBS: on_success(permit)
        ResilientClient --> CachedClient: Answer
        CachedClient --> Caller: Answer
    else Breaker Open
        CBP --> ResilientClient: Fast Fail (Open)
        ResilientClient -> CBS: Direct Failover
        CBS --> ResilientClient: Permit
        ResilientClient -> S: HTTP Request
        S --> ResilientClient: 200 OK
        ResilientClient --> Caller: Answer
    end
end
@enduml
```
</details>

---

## 3. Máy trạng thái Circuit Breaker Cửa sổ Trượt

Trạng thái và logic chuyển đổi của `SlidingWindowBreaker` gồm 3 trạng thái chuẩn: **CLOSED**, **OPEN**, và **HALF_OPEN**.

### Mermaid Diagram
```mermaid
stateDiagram-v2
    [*] --> CLOSED : Khởi tạo với window_size=10, min_calls=5

    state CLOSED {
        [*] --> Monitoring
        Monitoring : Cho phép 100% cuộc gọi đi qua
        Monitoring : Ghi nhận kết quả vào Circular Buffer
        Monitoring : Tính tỷ lệ lỗi failure_rate & slow_rate
    }

    state OPEN {
        [*] --> FastFailing
        FastFailing : Từ chối tức thì mọi cuộc gọi (CircuitOpenError)
        FastFailing : Không gọi mạng, bảo vệ nhà cung cấp
        FastFailing : Đếm thời gian trôi qua dựa vào virtual time 'now'
    }

    state HALF_OPEN {
        [*] --> Probing
        Probing : Chỉ cấp phép tối đa half_open_max_probes cuộc gọi
        Probing : Các cuộc gọi vượt số lượng probe bị từ chối
        Probing : Đếm số lần thành công liên tiếp
    }

    CLOSED --> OPEN : Điều kiện kích hoạt:\ntổng calls >= min_calls VÀ\n(failure_rate >= 0.5 HOẶC slow_rate >= 0.6)
    
    OPEN --> HALF_OPEN : Thời gian chờ kết thúc:\nnow >= last_state_change + open_duration_s (10s)
    
    HALF_OPEN --> CLOSED : Phục hồi hoàn toàn:\nĐạt đủ half_open_successes (2 lần) thành công liên tiếp\n[Reset Circular Buffer]
    
    HALF_OPEN --> OPEN : Tái phát sự cố:\nBẤT KỲ probe nào gặp lỗi / slow\n[Gia hạn mở mạch thêm 10s]
```

<details>
<summary><b>Nhấn để xem mã nguồn PlantUML</b></summary>

```plantuml
@startuml
[*] --> CLOSED : Khởi tạo

state CLOSED {
    CLOSED : Cho phép tất cả request
    CLOSED : Lưu metrics vào circular buffer
}

state OPEN {
    OPEN : Fail-fast mọi request
    OPEN : Ném CircuitOpenError
}

state HALF_OPEN {
    HALF_OPEN : Chỉ cấp phép thăm dò giới hạn
    HALF_OPEN : (half_open_max_probes)
}

CLOSED --> OPEN : calls >= min_calls AND\n(failure_rate >= 50% OR slow_rate >= 60%)
OPEN --> HALF_OPEN : now - opened_at >= open_duration_s
HALF_OPEN --> CLOSED : consecutive_successes >= target\n[Xóa trắng buffer]
HALF_OPEN --> OPEN : Bất kỳ probe nào thất bại\n[Reset timer open_duration_s]
@enduml
```
</details>

---

## 4. Luồng Xử lý Workflow Hoàn tiền Chính xác Một Lần — Exactly-Once

Quy trình xử lý hoàn tiền 6 bước bền vững (`RefundRequest`) có checkpointing tại từng bước và sinh **Idempotency Key** theo định dạng `session_id:step`.

### Mermaid Diagram
```mermaid
sequenceDiagram
    autonumber
    actor Caller as Hệ thống gọi Workflow
    participant Exec as Durable WorkflowExecutor
    participant Store as DurableStore (KV Store)
    participant Policy as Policy Service (Read-only)
    participant LLM as LLM Judge (Read-only)
    participant Ledger as Ledger Service (Side-effect)
    participant Gateway as Payment Gateway (Side-effect)
    participant Email as Email Service (Side-effect)
    participant Audit as Audit Service (Side-effect)

    Caller->>Exec: run(refund_request)
    
    %% Kiểm tra phục hồi trạng thái
    Exec->>Store: get(session_id + ":status")
    alt Workflow đã hoàn thành trước đó
        Store-->>Exec: "COMPLETED"
        Exec-->>Caller: Trả về kết quả hoàn tất ngay lập tức (Idempotent)
    end

    %% Step 1: Lookup Policy
    rect rgb(240, 248, 255)
    Note over Exec, Policy: BƯỚC 1: Tra cứu chính sách (Pure Read-only)
    Exec->>Store: get(session_id + ":step_lookup_policy")
    alt Đã có checkpoint
        Store-->>Exec: policy_data
    else Chưa có checkpoint
        Exec->>Policy: lookup_policy(request.order_id)
        Policy-->>Exec: policy_rules
        Exec->>Store: put(session_id + ":step_lookup_policy", policy_rules)
    end
    end

    %% Step 2: Call LLM Judge
    rect rgb(240, 248, 255)
    Note over Exec, LLM: BƯỚC 2: Đánh giá hợp lệ qua LLM (Pure Read-only)
    Exec->>Store: get(session_id + ":step_call_llm_judge")
    alt Đã có checkpoint
        Store-->>Exec: judge_decision
    else Chưa có checkpoint
        Exec->>LLM: evaluate(policy_rules, request.reason)
        LLM-->>Exec: {"approved": true, "max_amount": 5000}
        Exec->>Store: put(session_id + ":step_call_llm_judge", judge_decision)
    end
    end

    %% Step 3: Ledger Debit
    rect rgb(255, 250, 240)
    Note over Exec, Ledger: BƯỚC 3: Ghi nợ sổ cái tài khoản (Side-effect)
    Exec->>Store: get(session_id + ":step_ledger_debit")
    alt Đã có checkpoint
        Store-->>Exec: ledger_receipt
    else Chưa có checkpoint
        Note over Exec: Khóa Idempotency Key = session_id:ledger_debit
        Exec->>Ledger: debit(amount, idem_key="session_id:ledger_debit")
        Ledger-->>Exec: debit_tx_id = "tx_9981"
        Exec->>Store: put(session_id + ":step_ledger_debit", debit_tx_id)
    end
    end

    %% Step 4: Gateway Refund
    rect rgb(255, 250, 240)
    Note over Exec, Gateway: BƯỚC 4: Hoàn tiền qua Cổng thanh toán (Side-effect)
    Exec->>Store: get(session_id + ":step_gateway_refund")
    alt Đã có checkpoint
        Store-->>Exec: gateway_receipt
    else Chưa có checkpoint
        Note over Exec: Khóa Idempotency Key = session_id:gateway_refund
        Exec->>Gateway: refund(amount, idem_key="session_id:gateway_refund")
        Gateway-->>Exec: refund_ref = "ref_1024"
        Exec->>Store: put(session_id + ":step_gateway_refund", refund_ref)
    end
    end

    %% Step 5: Send Email
    rect rgb(255, 250, 240)
    Note over Exec, Email: BƯỚC 5: Gửi thông báo cho khách hàng (Side-effect)
    Exec->>Store: get(session_id + ":step_send_email")
    alt Đã có checkpoint
        Store-->>Exec: email_receipt
    else Chưa có checkpoint
        Note over Exec: Khóa Idempotency Key = session_id:send_email
        Exec->>Email: send(to, content, idem_key="session_id:send_email")
        Email-->>Exec: email_status = "sent"
        Exec->>Store: put(session_id + ":step_send_email", email_status)
    end
    end

    %% Step 6: Record Audit
    rect rgb(255, 250, 240)
    Note over Exec, Audit: BƯỚC 6: Ghi nhật ký kiểm toán (Side-effect)
    Exec->>Store: get(session_id + ":step_record_audit")
    alt Đã có checkpoint
        Store-->>Exec: audit_receipt
    else Chưa có checkpoint
        Note over Exec: Khóa Idempotency Key = session_id:record_audit
        Exec->>Audit: log_audit(session_id, idem_key="session_id:record_audit")
        Audit-->>Exec: audit_ack = "logged"
        Exec->>Store: put(session_id + ":step_record_audit", audit_ack)
    end
    end

    %% Đánh dấu hoàn tất
    Exec->>Store: put(session_id + ":status", "COMPLETED")
    Exec-->>Caller: Hoàn thành WorkflowOutcome (success=True)
```

---

## 5. Cơ chế Phục hồi khi Gặp Crash & Idempotency

Sơ đồ khối thể hiện cách `execute_step` xử lý khi tiến trình bị sập nguồn (Crash) hoặc gặp sự cố mạng (ServiceTimeout / 500 error).

### Mermaid Diagram
```mermaid
flowchart TD
    Start([Bắt đầu thực hiện Bước i]) --> CheckStore{DurableStore đã lưu\ncheckpoint bước i chưa?}
    
    CheckStore -->|Đã có kết quả| UseCached[Tái sử dụng kết quả đã checkpoint\nBỏ qua không gọi lại dịch vụ]
    UseCached --> NextStep([Chuyển sang Bước tiếp theo])

    CheckStore -->|Chưa có| GenKey[Sinh Idempotency Key:\nsession_id + ':' + step_name]
    GenKey --> CallService[Gọi Dịch vụ bên ngoài với idem_key]

    CallService --> ResultEval{Kết quả thực thi}

    ResultEval -->|Crash trước khi gửi yêu cầu| CrashBefore[Tiến trình bị hủy]
    CrashBefore -.->|Worker khởi động lại| Start

    ResultEval -->|Timeout / Lỗi mạng| HandleTimeout[Bắt ngoại lệ ServiceTimeout]
    HandleTimeout --> RetryStep[Thử lại chính xác với idem_key cũ]
    RetryStep --> CallService

    ResultEval -->|Dịch vụ xử lý xong & Crash trước khi Checkpoint| CrashMid[Tiến trình bị hủy]
    CrashMid -.->|Worker khởi động lại| Start
    NoteMid[Dịch vụ nhận lại idem_key cũ ->\nTrả lại kết quả ban đầu mà KHÔNG trừ tiền lần 2]
    CrashMid --- NoteMid

    ResultEval -->|Thành công trả về kết quả| SaveStore[Lưu checkpoint vào DurableStore:\nstore.put session_id:step, result]
    SaveStore --> NextStep
```

<details>
<summary><b>Nhấn để xem mã nguồn PlantUML</b></summary>

```plantuml
@startuml
start
:Bắt đầu thực hiện Step;
if (DurableStore đã có checkpoint cho step?) then (yes)
  :Lấy kết quả đã lưu;
  :Bỏ qua thực thi mạng (Zero-cost replay);
else (no)
  :Tạo Idempotency Key = session_id:step_name;
  repeat
    :Gọi dịch vụ bên ngoài kèm Idempotency Key;
    backward: Backoff ngắn & Thử lại;
  repeat while (Gặp ServiceTimeout hoặc lỗi tạm thời?) is (yes)
  -> no (Thành công);
  :Ghi checkpoint kết quả vào DurableStore;
endif
:Tiếp tục bước tiếp theo;
stop
@enduml
```
</details>

---

## 6. Công cụ Cảnh báo SRE Multi-Window Multi-Burn-Rate

Quy trình đánh giá chuỗi thời gian 43.200 phút giả lập sử dụng **Prefix Sums** $O(1)$ và 4 bộ quy tắc đa cửa sổ (Multi-Window Multi-Burn-Rate) theo chuẩn Google SRE.

### Mermaid Diagram
```mermaid
sequenceDiagram
    autonumber
    participant Engine as Test Runner / SRE Engine
    participant WS as WindowSums (Prefix Sums O(1))
    participant Alerter as MultiWindowAlerter
    participant SRE as Đội ngũ SRE trực ban

    Engine->>WS: build(total_requests_per_min, bad_requests_per_min)
    Note over WS: Tiền tính toán mảng cộng dồn Prefix Sums P[t]:\nSum[L, R] = P[R] - P[L - 1] trong thời gian O(1)

    Engine->>Alerter: evaluate(total, bad)
    
    loop Duyệt từng phút t từ 0 đến 43.199 (30 ngày)
        
        %% Rule 1
        Note over Alerter, WS: Kiểm tra Tier 1 (Page khẩn cấp): Cửa sổ dài 1h (60m) & Cửa sổ ngắn 5m
        Alerter->>WS: query_rate(t - 60, t) & query_rate(t - 5, t)
        WS-->>Alerter: error_rate_1h, error_rate_5m
        opt Cả 2 cửa sổ đều vượt Burn Rate 14.4x
            Alerter->>SRE: Kích hoạt ALERT_PAGE (Mất 2% budget trong 1 giờ)
        end

        %% Rule 2
        Note over Alerter, WS: Kiểm tra Tier 2 (Page nghiêm trọng): Cửa sổ dài 6h (360m) & Cửa sổ ngắn 30m
        Alerter->>WS: query_rate(t - 360, t) & query_rate(t - 30, t)
        WS-->>Alerter: error_rate_6h, error_rate_30m
        opt Cả 2 cửa sổ đều vượt Burn Rate 6.0x
            Alerter->>SRE: Kích hoạt ALERT_PAGE (Mất 5% budget trong 6 giờ)
        end

        %% Rule 3
        Note over Alerter, WS: Kiểm tra Tier 3 (Ticket cảnh báo): Cửa sổ dài 24h (1440m) & Cửa sổ ngắn 2h (120m)
        Alerter->>WS: query_rate(t - 1440, t) & query_rate(t - 120, t)
        WS-->>Alerter: error_rate_24h, error_rate_2h
        opt Cả 2 cửa sổ đều vượt Burn Rate 3.0x
            Alerter->>SRE: Kích hoạt TICKET (Mất 10% budget trong 1 ngày)
        end

        %% Rule 4
        Note over Alerter, WS: Kiểm tra Tier 4 (Ticket xu hướng): Cửa sổ dài 72h (4320m) & Cửa sổ ngắn 6h (360m)
        Alerter->>WS: query_rate(t - 4320, t) & query_rate(t - 360, t)
        WS-->>Alerter: error_rate_72h, error_rate_6h
        opt Cả 2 cửa sổ đều vượt Burn Rate 1.0x
            Alerter->>SRE: Kích hoạt TICKET (Mất 10% budget trong 3 ngày)
        end
    end

    Alerter->>Alerter: Gom các phút liên tục thành AlertEvent(tier, start_min, end_min)
    Alerter-->>Engine: Trả về danh sách Sự kiện Cảnh báo
```

---

## 7. Sơ đồ Lớp Toàn diện

Biểu đồ lớp thể hiện toàn bộ các thực thể, thuộc tính và phương thức cốt lõi được hiện thực trong gói `src/gameday_lab/`.

### Mermaid Diagram
```mermaid
classDiagram
    class BaseClient {
        <<abstract>>
        +send(call, now)* Action
        +on_success(call, tag, latency, now)* Action
        +on_failure(call, tag, err, now)* Action
        +on_cancel(call, tag, now)* None
    }

    class ResilientClient {
        -ladder: list[str]
        -breakers: dict[str, SlidingWindowBreaker]
        -budget: RetryBudget
        -backoff: FullJitterBackoff
        -cooldown_until: dict[str, float]
        -min_time_left_s: float
        +send(call, now) Action
        +on_success(call, tag, latency, now) Action
        +on_failure(call, tag, err, now) Action
        -_route(call, now, delay) Action
        -_send(call, provider, now, delay) Action
    }

    class CachedClient {
        -inner: BaseClient
        -cache: dict[str, CacheEntry]
        -similarity_threshold: float
        -ttl_s: float
        -max_stale_s: float
        +send(call, now) Action
        +on_success(call, tag, latency, now) Action
        +on_failure(call, tag, err, now) Action
        -_find_match(query_emb) CacheEntry
    }

    class SlidingWindowBreaker {
        +name: str
        +window_size: int
        +min_calls: int
        +failure_rate_threshold: float
        +slow_call_s: float
        +slow_rate_threshold: float
        +open_duration_s: float
        +half_open_max_probes: int
        +half_open_successes: int
        -state: BreakerState
        -buffer: list[CallOutcome]
        +try_acquire(now) Permit
        +on_success(permit, latency, now) None
        +on_failure(permit, now) None
    }

    class RetryBudget {
        +retry_ratio: float
        +window_s: float
        +min_retry_rate: float
        -history: deque
        +try_acquire_retry(now) bool
        +record_success(now) None
        +record_attempt(now) None
    }

    class FullJitterBackoff {
        +base_s: float
        +factor: float
        +max_s: float
        +delay(attempt) float
    }

    class DurableStore {
        <<interface>>
        +get(key) Any
        +put(key, value) None
    }

    class WorkflowExecutor {
        -store: DurableStore
        -services: Services
        +run(request) WorkflowOutcome
        -_execute_step(request, step_name, fn) Any
    }

    class MultiWindowAlerter {
        +rules: list[AlertRule]
        +slo_target: float
        +evaluate(total, bad) list[AlertEvent]
    }

    class WindowSums {
        -prefix_total: list[int]
        -prefix_bad: list[int]
        +sum_total(start, end) int
        +sum_bad(start, end) int
        +error_rate(start, end) float
    }

    BaseClient <|-- ResilientClient
    BaseClient <|-- CachedClient
    ResilientClient o-- SlidingWindowBreaker
    ResilientClient o-- RetryBudget
    ResilientClient o-- FullJitterBackoff
    CachedClient o-- BaseClient
    WorkflowExecutor o-- DurableStore
    MultiWindowAlerter o-- WindowSums
```

<details>
<summary><b>Nhấn để xem mã nguồn PlantUML</b></summary>

```plantuml
@startuml
interface BaseClient {
    +send(call, now): Action
    +on_success(call, tag, latency, now): Action
    +on_failure(call, tag, err, now): Action
}

class ResilientClient {
    -ladder: List<String>
    -breakers: Map<String, SlidingWindowBreaker>
    -budget: RetryBudget
    -backoff: FullJitterBackoff
    +send(call, now): Action
    +on_success(call, tag, latency, now): Action
    +on_failure(call, tag, err, now): Action
}

class CachedClient {
    -inner: BaseClient
    -similarity_threshold: float
    -ttl_s: float
    -max_stale_s: float
    +send(call, now): Action
}

class SlidingWindowBreaker {
    -state: BreakerState
    -buffer: List<CallOutcome>
    +try_acquire(now): Permit
    +on_success(permit, latency, now)
    +on_failure(permit, now)
}

class RetryBudget {
    -retry_ratio: float
    -window_s: float
    +try_acquire_retry(now): bool
    +record_success(now)
}

class WorkflowExecutor {
    -store: DurableStore
    +run(request: RefundRequest): WorkflowOutcome
}

interface DurableStore {
    +get(key: String): Object
    +put(key: String, val: Object)
}

class MultiWindowAlerter {
    +evaluate(total, bad): List<AlertEvent>
}

class WindowSums {
    -prefix_total: List<int>
    -prefix_bad: List<int>
    +error_rate(start, end): float
}

BaseClient <|-- ResilientClient
BaseClient <|-- CachedClient
CachedClient o-- BaseClient
ResilientClient *-- SlidingWindowBreaker
ResilientClient *-- RetryBudget
WorkflowExecutor o-- DurableStore
MultiWindowAlerter o-- WindowSums
@enduml
```
</details>

---

## 8. Hướng dẫn Sử dụng & Trực quan hóa

1. **Xem trực tiếp trong IDE / VS Code:**
   - Mở file `WORKFLOW_UML.md`.
   - Nhấn tổ hợp phím `Ctrl + Shift + V` (hoặc click icon Preview ở góc phải trên) để mở Markdown Preview. Toàn bộ các diagram Mermaid sẽ tự động render dạng đồ họa tương tác.
2. **Xem dạng PlantUML:**
   - Nếu máy tính có cài extension PlantUML trong VS Code, mở khối code trong phần `<details>` tương ứng và nhấn `Alt + D` để render sơ đồ PlantUML.
