## Blog of highland0971

### 基于无IP list的GFW IP解锁

在以往的基于路由、iptables解锁GFW IP封锁的方法中，均需要一份国内或国外的IP地址列表，然后进行策略路由。基于这种方案的解锁，需要人工去维护一张IP地址列表，其中基于[chnroutes](https://github.com/fivesheep/chnroutes)方案生成的国内IP地址列表就有8000多条记录,在加载的时候会比较耗时，同时给路由器带来一定负担。

本文给出一种基于无IP列表的自动IP访问监测及路由更变策略，详情如下：

```markdown
Syntax highlighted code block

# Header 1
## Header 2
### Header 3

- iptables -I FORWARD -i br0 -o eth0 -p tcp -m tcp --tcp-flags SYN,ACK SYN -m recent --rcheck --seconds 15 --hitcount 3 --name syn_watch_list --rdest -j LOG --log-prefix "GFW_DECT_SYN "
- iptables -I FORWARD -i br0 -o eth0 -p tcp -m tcp --tcp-flags SYN,ACK SYN -m recent --set --name syn_watch_list --rdest
- iptables -I FORWARD -i br0 -o eth0 -p tcp -m tcp --tcp-flags SYN,ACK SYN -j LOG --log-prefix "GFW_DEBUG "
- iptables -I FORWARD -i eth0 -p tcp -m tcp --tcp-flags RST RST -j LOG --log-prefix "GFW_DECT_RST "


- Bulleted
- List

1. Numbered
2. List

**Bold** and _Italic_ and `Code` text

[Link](url) and ![Image](src)
```

For more details see [GitHub Flavored Markdown](https://guides.github.com/features/mastering-markdown/).

### Jekyll Themes

Your Pages site will use the layout and styles from the Jekyll theme you have selected in your [repository settings](https://github.com/highland0971/highland0971.github.io/settings). The name of this theme is saved in the Jekyll `_config.yml` configuration file.

### Support or Contact

Having trouble with Pages? Check out our [documentation](https://help.github.com/categories/github-pages-basics/) or [contact support](https://github.com/contact) and we’ll help you sort it out.
