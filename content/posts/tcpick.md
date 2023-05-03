---
type: "post"
title: "TCPick"
date: 2022-03-25T18:13:19+03:00
draft: false
tags: ["networks", "tcpick"]
---

Чтобы не вылезая из консоли посмотреть содержимое дампа трафика в открытом виде можно использовать tcpick.

```shell
tcpick -C -yP -r tcp_dump.pcap
```
Извлечь содержимое дампа без Wireshark тоже можно не вставая с дивана, но уже другой утилитой - tcpxtract.

```shell
tcpxtract -f tcp_dump.pcac
```

