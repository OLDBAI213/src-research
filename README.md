# AI Agent SRC 漏洞挖掘研究

> AI Agent从零学习SRC漏洞挖掘的全过程记录 — 研究笔记、框架分析、真实案例、学习路线

## 研究成果

### 方法论
- **SRC工作方法论v2** — 八步流程（工具检查→最大化线索→分析薄弱→价值评估→执行攻击→质量门禁→深入升级→报告撰写→持续积累）
- **7-Question Gate** — 报告前质量门禁（一个不过就杀掉）
- **永不提交清单** — 30+条SRC适用避坑指南

### 工具研究
- 9个AI安全工具深度研究（Claude-BugHunter/Shannon/Strix/DeepAudit等）
- 8个Hermes内置安全技能整合
- 15个Claude-BugHunter核心技能（共300K+字符）

### 实战记录
- 朴朴超市SRC项目完整记录（6个漏洞已提交）
- 攻击记忆库35条（覆盖9个项目+外部学习）
- 复用策略5个（配置泄露/多端分裂/前端JS情报/四层拆解/业务逻辑）

### 框架知识库
- Spring Boot危险模式（SpEL注入/Actuator未授权/反序列化RCE）
- Laravel危险模式（Blade模板注入/.env泄露/PHAR反序列化）
- ThinkPHP危险模式（5.x RCE/order by注入/POP链）

## 目录

```
docs/
├── SRC工作方法论.md
├── SRC项目记录-朴朴超市.md
├── 攻击记忆系统设计.md
├── AI黑客工具研究-8个项目.md
├── SRC平台目录-完整版.md
├── 靶场平台研究.md
├── 框架知识库/
│   ├── spring-boot.json
│   ├── laravel.json
│   └── thinkphp.json
└── ...
```

## 更新日志

- 2026-05-27: 完成9个AI工具研究+八步方法论+攻击记忆系统+框架知识库
- 2026-05-25: 朴朴SRC项目完成6个漏洞提交
- 2026-05-21: 初始研究计划建立

## License

MIT
