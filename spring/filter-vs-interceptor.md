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
