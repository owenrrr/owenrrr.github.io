---
layout: post
title:  "Prompt Injection"
date:   2026-03-09 19:35:55 +0800
categories: Demo
feature-img: "https://www.paloaltonetworks.com/content/dam/pan/en_US/images/cyberpedia/what-is-a-prompt-injection-attack/GenAI-Security-2025_6-1.png"
img: "https://www.paloaltonetworks.com/content/dam/pan/en_US/images/cyberpedia/what-is-a-prompt-injection-attack/GenAI-Security-2025_6-1.png"
tags: Cybersecurity
---
<html>
<body>
<div markdown="block" style="margin-top: 10px">
    
## Introduction:
Here's the list to provide some materials describing the basic concept of prompt injection, the difference between jailbreaking and prompt injection, and the real-world examples of prompt injection. 

## Articles & Papers
- [AI Under Attack: Six Key Adversarial Attacks and Their Consequences](https://mindgard.ai/blog/ai-under-attack-six-key-adversarial-attacks-and-their-consequences)

This article introduces six mainstraem AI cybersecurity attacks with their corresponding consequences and remediations. Among them, prompt injection is placed on the first.

- [IBM - What is a prompt injection attack?](https://www.ibm.com/think/topics/prompt-injection)

***A prompt injection is a type of cyberattack against large language models (LLMs). Hackers disguise malicious inputs as legitimate prompts, manipulating generative AI systems (GenAI) into leaking sensitive data, spreading misinformation, or worse.***

In this IBM's article, it also provides several prevention and mitigation strategies to prompt injection cyberattack, including general input validation, least priviledge policy, and human-in-the-loop on high risk actions.

- [Palo Alto - What Is a Prompt Injection Attack?](https://www.paloaltonetworks.com/cyberpedia/what-is-a-prompt-injection-attack)

This article describes the basic concept of prompt injection and its type including direct prompt injection and indirect prompt injection. It also explains the difference between jailbreaking and prompt injection, which is their goals and processes.

***Again, prompt injection happens when an attacker embeds malicious instructions into an AI's input field to override its original programming. So the goal is to make the AI ignore prior instructions and follow the attacker’s command instead.***

***On the other hand: Jailbreaking is a technique used to remove or bypass an AI system’s built-in safeguards. The goal is to make the model ignore its ethical constraints or safety filters.***

- [PLeak: Prompt Leaking Attacks against Large Language Model Applications](https://arxiv.org/abs/2405.06823)

In this paper, it designs a novel, closed-box prompt leaking attack framework, called PLeak, to optimize an adversarial query such that when the attacker sends it to a target LLM application, its response reveals its own system prompt.

<br>

## Remediation & Mitigation
![Palo Alto Prompt Injection Solutions](https://www.paloaltonetworks.com/content/dam/pan/en_US/images/cyberpedia/what-is-a-prompt-injection-attack/Prompt-injection-attack-2025_7.png?imwidth=1920)

</div>
</body>
</html>
