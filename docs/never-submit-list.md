# 永不提交清单（SRC适用版）

**最后更新**: 2026-05-27
**来源**: Claude-BugHunter triage-validation + 国内SRC平台经验

提交这些会毁掉你的有效率。一个不过 = 杀掉。

## 绝对不提交

- 缺少CSP/HSTS/安全头
- 缺少SPF/DKIM/DMARC
- GraphQL内省（无认证绕过/IDOR证明）
- Banner/版本泄露（无可用CVE利用）
- 非敏感页面的点击劫持（无敏感操作PoC）
- Tabnabbing
- CSV注入（无实际代码执行证明）
- CORS通配符(*)（无凭据泄露PoC）
- 登出CSRF
- 自XSS（只影响自己账号）
- 开放重定向（无ATO或OAuth盗窃链）
- 移动端OAuth client_secret（已知，预期行为）
- SSRF仅DNS回调（无内部服务访问或数据）
- Host头注入（无密码重置投毒PoC）
- 非关键表单的速率限制（搜索/联系/有Cloudflare的登录）
- 登出后Session不失效
- 并发会话
- 错误消息中的内网IP
- 混合内容
- 弱SSL密码套件
- 缺少HttpOnly/Secure cookie标志（单独）
- 失效的外部链接
- 密码字段的自动填充
- 预账号接管（通常——需要非常特殊条件）

## 需要链式组合才有效

| 独立发现 | 需要链接 | 有效结果 |
|---|---|---|
| 开放重定向 | + OAuth redirect_uri | ATO (Critical) |
| 点击劫持 | + 敏感操作 + PoC | Medium |
| CORS通配符 | + 凭据请求泄露PII | High |
| CSRF | + 敏感操作 | High |
| 速率限制绕过 | + OTP/重置Token暴力 | Medium/High |
| SSRF仅DNS | + 内部服务+数据 | Medium |
| Host头注入 | + 密码重置投毒 | High |
| 自XSS | + CSRF触发 | Medium |
| 子域接管 | + OAuth redirect_uri | Critical |
| GraphQL内省 | + 认证绕过/IDOR | High |

## 持续补充

发现新的"永不提交"类型或链式组合，追加到这里。
