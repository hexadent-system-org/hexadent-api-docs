# 公开 Error Code 和前端行为

本文档定义 Web、iOS 和 Android 客户端需要明确处理的公开 API Error Code。

API 负责提供面向用户的英文错误提示。客户端不得解析 `message`，也不得根据 `message` 决定业务流程。客户端只使用 `error.code` 执行本文档定义的行为；对于其他错误，直接展示 `error.message`。

## 错误响应结构

```json
{
  "error": {
    "code": "REQUEST_VALIDATION_ERROR",
    "message": "Please check the highlighted fields and try again.",
    "requestId": "01K...",
    "fieldErrors": [
      {
        "field": "countryCode",
        "message": "Select a supported country."
      }
    ],
    "details": null
  }
}
```

- `code` 是稳定的公开错误码，用于决定客户端行为。
- `message` 是由 API 负责维护、可以安全展示给用户的英文提示。
- `requestId` 用于关联本次错误的客户端日志、服务端日志和支持请求。
- 只有存在字段级校验错误时才返回 `fieldErrors`。
- `details` 是可选的客户端安全结构化信息；没有内容时可以省略或返回 `null`。它不用于字段校验错误。
- 内部异常类型、堆栈和数据库错误等诊断信息不得放入 `details`，也不属于公开 API Contract。

## 需要前端明确处理的 Code

| 公开 Error Code | HTTP 状态 | 用户提示 | 前端必须执行的行为 |
| --- | ---: | --- | --- |
| `UNAUTHORIZED` | 401 | Your session has expired. Please sign in again. | 尝试刷新 Session 一次。如果刷新失败，清除 Session 并进入登录页。不得循环重试。 |
| `EMAIL_NOT_VERIFIED` | 403 | Please verify your email address to continue. | 进入邮箱验证页面，并提供重新发送验证邮件的操作。 |
| `ONBOARDING_REQUIRED` | 403 | Let’s finish setting up your workspace. | 直接进入 Workspace Setup，通常不需要显示错误 Toast。 |
| `ACCESS_RESTRICTED` | 403 | Your access to this organization is currently restricted. Contact your organization administrator or support for help. | 显示 Access Restricted 页面，不得自动重试。 |
| `ACCOUNT_SELECTION_REQUIRED` | 409 | Choose an organization to continue. | 打开 Organization Selector。 |
| `REQUEST_VALIDATION_ERROR` | 422 | Please check the highlighted fields and try again. | 将 `fieldErrors` 映射到对应表单控件；无法映射的错误显示在表单级错误区域。 |

## 其他错误的默认行为

对于上表未列出的 Error Code，客户端统一执行以下规则：

1. 展示 API 返回的 `error.message`。
2. 将 `requestId` 写入客户端诊断日志，并在支持请求中携带该值。
3. 如果 API 没有返回 `message`，显示：**Something went wrong. Please try again.**

客户端遇到未知 Error Code 时必须使用上述默认行为，不能因此崩溃。

## 字段校验错误

每个字段错误采用以下公开结构：

```ts
type FieldError = {
  field: string;
  message: string;
};
```

- `field` 使用请求 JSON 中公开的 camelCase 字段名，例如 `countryCode` 或 `addressLine1`。
- 客户端在对应表单控件旁显示 `message`。
- 如果找不到对应字段，客户端应在表单级错误区域显示该提示。
- 同一个字段存在多个错误时，客户端默认显示第一条。

## 前端处理示例

```ts
function handleApiError(error: ApiError) {
  switch (error.code) {
    case "UNAUTHORIZED":
      return refreshSessionOrSignIn();
    case "EMAIL_NOT_VERIFIED":
      return navigateToEmailVerification();
    case "ONBOARDING_REQUIRED":
      return navigateToOnboarding();
    case "ACCESS_RESTRICTED":
      return showAccessRestricted(error.message);
    case "ACCOUNT_SELECTION_REQUIRED":
      return navigateToOrganizationSelector();
    case "REQUEST_VALIDATION_ERROR":
      return applyFieldErrors(error.fieldErrors);
    default:
      return showError(
        error.message || "Something went wrong. Please try again.",
      );
  }
}
```

以上代码仅用于说明处理规则。Web、iOS 和 Android 可以根据各自的导航及状态管理架构调整具体实现，但必须保持本文档定义的行为一致。
