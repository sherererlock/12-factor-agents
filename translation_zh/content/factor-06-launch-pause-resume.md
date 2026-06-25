[← 返回 README](https://github.com/humanlayer/12-factor-agents/blob/main/README.md)

### 6. 通过简单 API 启动/暂停/恢复

智能体本质上就是程序，我们对如何启动、查询、恢复和停止它们有明确的期望。

[![pause-resume animation](https://github.com/humanlayer/12-factor-agents/blob/main/img/165-pause-resume-animation.gif)](https://github.com/user-attachments/assets/feb1a425-cb96-4009-a133-8bd29480f21f)

<details>
<summary><a href="https://github.com/humanlayer/12-factor-agents/blob/main/img/165-pause-resume-animation.gif">GIF 版本</a></summary>

![pause-resume animation](https://github.com/humanlayer/12-factor-agents/blob/main/img/165-pause-resume-animation.gif)

</details>


用户、应用程序、流水线和其他智能体应该能够通过简单的 API 轻松启动一个智能体。

智能体及其编排的确定性代码应该能够在需要长时间运行的操作时暂停智能体。

Webhook 等外部触发器应该能够让智能体从上次中断的地方恢复，而无需与智能体编排器进行深度集成。

与[因素 5 - 统一执行状态和业务状态](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-05-unify-execution-state.md)和[因素 8 - 掌控你的控制流](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-08-own-your-control-flow.md)密切相关，但可以独立实现。



**注意** —— 通常 AI 编排器会允许暂停和恢复，但不允许在工具选择和工具执行之间进行。另请参见[因素 7 - 通过工具调用联系人类](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-07-contact-humans-with-tools.md)和[因素 11 - 从任何地方触发，在用户所在之处与他们会合](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-11-trigger-from-anywhere.md)。

[← 统一执行状态](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-05-unify-execution-state.md) | [通过工具调用联系人类 →](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-07-contact-humans-with-tools.md)
