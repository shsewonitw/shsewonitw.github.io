---
layout: post-backup
title: "[Debug] HTTPS API Call SSLHandshakeException 발생"
subtitle: "SSLHandshakeException"
category: devlog
tags: debug
---

![image](https://user-images.githubusercontent.com/50475160/169509674-240d6772-c86a-4abe-88bb-e070507ffd54.png)


> :pencil:

자바에서 Https Call 을 할때 발생하는 오류

```java
javax.net.ssl.SSLHandshakeException: sun.security.validator.ValidatorException: PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target

```

