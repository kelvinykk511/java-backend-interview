# Filter vs Interceptor

## 面試問題

Spring MVC 裡 Filter 和 Interceptor 差在哪？認證、包裝 Request、改 Response 各適合放哪裡？

## 答法

- **Filter（Servlet 規格）**：在 DispatcherServlet **之外**；整個鏈是「進容器 → Filter 鏈 → Servlet → 回 Filter」。能包 `HttpServletRequest`／`Response`、早退、做 CORS／壓縮／全域認證閘門。
- **Interceptor（Spring MVC）**：在 DispatcherServlet **之內**；掛在 Handler 執行前後（`preHandle`／`postHandle`／`afterCompletion`），能拿到 Handler／ModelAndView，適合登入態、權限、統一日誌、開銷計量。
- **順序直覺**：Filter（外）→ DispatcherServlet → Interceptor → Controller。
- **選法**：要改底層 Request／Response 或與 Spring 無關的通用能力 → Filter；要跟 Controller 映射、業務上下文綁在一起 → Interceptor。
- **面試一句話**：Filter 偏容器級、更外側；Interceptor 偏 Spring MVC 級、能碰 Handler。別混著說「都是 AOP」。

## 常見追問／陷阱

「Interceptor 裡包一層 RequestWrapper 行不行？」

→ 多數場景太晚：前面 Filter／框架可能已讀過 body。要包裝 body／字元集，放 **Filter** 更穩。另一陷阱：`preHandle` 回 `false` 不會進 Controller，但已執行的 Filter 仍會跑完；清 ThreadLocal 要放 `afterCompletion` 或 Filter 的 `finally`。

## 小練習

要做「簽名驗證 + 讀 body 重放防護 + 寫入 userId 到請求屬性給 Controller」。簽名驗證放 Filter 還是 Interceptor？userId 屬性呢？

---

## 深入（2026-11-15）：責任鏈位置、OncePerRequestFilter、同 Security Filter

### 面試可以點追

下單網關要驗 HMAC（讀 raw body）、防重放，再把 userId 傳去 Controller；中間仲有 Spring Security。呢啲放 Filter 定 Interceptor？`OncePerRequestFilter` 解決咩問題？

### 答法

**位置先決定能力**

- Filter 喺 DispatcherServlet **之外**（Servlet 容器責任鏈）：可以包 Request／Response、早退、做 CORS／壓縮／全域閘門。**Spring Security 本身都係 Filter**（`DelegatingFilterProxy`），比任何 Interceptor 都早。
- Interceptor 喺 DispatcherServlet **之內**：要有 Handler／`HandlerMapping` 先會跑；拿得到 `HandlerMethod`，適合權限細節、統一 access log、耗時。404、靜態資源、唔經過 DispatcherServlet 嘅路徑，Interceptor 可能根本唔入。
- 順序：Filter（含 Security）→ DispatcherServlet → Interceptor → Controller。

**讀 body 驗簽一定要 Filter**

- HMAC 要原始字節，`InputStream` 只能讀一次，必須喺最外層用 `ContentCachingRequestWrapper`／自訂 wrapper 包住再傳下去。放到 Interceptor 通常太晚：前面 Filter／框架可能已讀過 body。
- 驗簽成功後用 `request.setAttribute`（或寫入 `SecurityContext`）傳 userId，Controller／後面的 Interceptor 再取。可以，但唔好只靠 ThreadLocal 當唯一通道（異步會斷、線程池會串數據）。

**OncePerRequestFilter**

- FORWARD／ERROR／ASYNC 派發可能令**同一請求再行一次 Filter 鏈**。再讀 body 會偶發驗簽失敗；限流 Filter 重跑會多扣一次配額。
- `OncePerRequestFilter` 用 request attribute 標記，同一請求只執行一次邏輯。ASYNC 派發要另計，搞清楚 `shouldNotFilterAsyncDispatch`。
- 註冊用 `FilterRegistrationBean`，先保證能注入 Spring Bean；純 `@WebFilter` 由容器直接起，可能早於 Spring 上下文，`@Autowired` 會係 null。

### 追問／陷阱

- `preHandle` 回 `false`：唔入 Controller，但已經執行嘅 Filter 仍會跑完；`afterCompletion` 仍可能被叫。清 ThreadLocal 要 `afterCompletion` **同** Filter `finally` 都做。
- 異步陷阱：`DeferredResult`／`CompletableFuture`／`@Async` 會換線程。Interceptor 喺 Tomcat 線程清嘅 ThreadLocal，傳唔到業務線程；線程池複用時仲可能串到上一個用戶。屬性放 request／SecurityContext，或用 TaskDecorator／TTL 明確傳遞。
- 唔好話「Filter 同 Interceptor 都係 AOP」。亦唔好用 Interceptor 取代 Spring Security。
