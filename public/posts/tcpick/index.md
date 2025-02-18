# TCPick


Чтобы не вылезая из консоли посмотреть содержимое дампа трафика в открытом виде можно использовать tcpick.
&lt;!--more--&gt;
```shell
tcpick -C -yP -r tcp_dump.pcap
```
Извлечь содержимое дампа без Wireshark тоже можно не вставая с дивана, но уже другой утилитой - tcpxtract.

```shell
tcpxtract -f tcp_dump.pcac
```



---

> Author:   
> URL: http://localhost:1313/posts/tcpick/  

